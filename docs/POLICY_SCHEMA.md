# Policy Schema

Status: Draft v0.1. The machine-readable schema is `schemas/policy.v1.schema.json`. If the two disagree, the JSON Schema is authoritative and this document must be fixed.

## 1. Principles

- **Default deny.** An action is allowed only if a grant explicitly matches it and every rule passes.
- **Explicit over implicit.** No wildcards in MVP for amounts, currencies, or beneficiaries. `*` is accepted only for `agents` in a grant that is itself time-limited and marked `broad: true`, which triggers a warning in the policy linter.
- **Immutable and hashed.** A policy version is never edited. A change creates a new version number. The policy hash is SHA-256 over the canonical JSON.
- **Monotonic.** Version numbers only increase. The gate refuses to activate a version lower than the highest activated.
- **Signed activation.** A policy becomes active only when a signed `policy_activation` receipt is recorded.

## 2. Top-level structure

```yaml
policy_version_schema: "1"
policy_id: ap-vendor-payments          # stable identifier across versions
policy_version: 14                      # monotonic integer
effective_from: "2026-11-01T00:00:00Z"
effective_until: null                   # optional
owner: { role: "finance-controller" }   # who is accountable for this policy
currencies: [GBP]                       # currencies this policy covers
agents: []                              # registered agents (section 3)
beneficiaries: []                       # allowlist (section 4)
approval_rules: []                      # thresholds and approver groups (section 5)
grants: []                              # what each agent may do (section 6)
```

## 3. Agents

```yaml
agents:
  - agent_id: agent-ap-bot-01
    owner_id: user-finance-lead-07       # must not be an approver for payments this agent initiates
    key_id: agent-key-2026-03
    public_key: "ed25519:..."
    status: active                       # active | suspended | revoked
    valid_from: "2026-11-01T00:00:00Z"
    valid_until: "2027-11-01T00:00:00Z"
```

Rules:
- `agent_id` unique within the policy.
- `key_id` must be unique across all active agents.
- Revoked agents remain listed so historical receipts stay verifiable.

## 4. Beneficiaries

```yaml
beneficiaries:
  - beneficiary_id: ben-acme-supplies
    account_hash: "sha256:4f1c0e..."     # salted hash of account details
    account_salt_ref: salt-2026-11
    name_on_record: "Acme Supplies Ltd"  # stored in customer system; not in receipts
    status: active
    added_by_approval: appr-2f11         # approval reference for adding the beneficiary
```

Rules:
- A payment to a beneficiary not on this list is DENY.
- Adding or changing a beneficiary requires an approval rule with `scope: beneficiary_change`.

## 5. Approval rules

```yaml
approval_rules:
  - rule_id: standard-payment
    applies_to: { action: payment.create, currency: GBP }
    thresholds:
      - above: 0                         # amounts in minor units
        required_approvals: 1
        approver_groups: [finance-approvers]
      - above: 1000000                   # £10,000
        required_approvals: 2
        approver_groups: [finance-approvers, finance-controllers]
        distinct_groups: true            # each approval must come from a different group
    separation_of_duties:
      approver_not_owner: true           # approver cannot be the owner of the initiating agent
      approver_not_initiator: true
```

Rules:
- Thresholds are evaluated from the highest matching `above` value downward.
- `required_approvals` must be satisfiable by the groups listed. A policy where it cannot be met is rejected by the linter.
- Approver membership is provided by an approver registry (see open decision 2 in `docs/SPEC.md`).

## 6. Grants

```yaml
grants:
  - grant_id: grant-ap-bot-01-vendors
    agent_id: agent-ap-bot-01
    action: payment.create
    currency: GBP
    per_transaction_limit: 500000        # £5,000 in minor units
    daily_cap: 2000000                   # £20,000
    rolling_window_cap:                  # optional anti-splitting control
      window: PT24H
      limit: 2000000
    beneficiaries: [ben-acme-supplies, ben-northwind-logistics]
    valid_window:
      days: [MON, TUE, WED, THU, FRI]
      start_time: "08:00"
      end_time: "18:00"
      timezone: Europe/London
    broad: false
```

Rules:
- A grant applies only to the agent, action, and currency it names.
- Caps are checked against the sum of already-authorised and consumed amounts in the window, read from the receipt log.
- If the receipt log cannot be read to compute caps, the request is DENY (fail closed).
- `broad: true` is allowed only with `valid_until` set on the grant. (Schema enforces this as a conditional requirement.)

## 7. Evaluation order

For each request, rules run in this fixed order. The first `NO_MATCH` or `ERROR` ends evaluation with DENY, but the receipt still records every rule that ran:

1. `agent.registered`: agent exists in the active policy and key matches.
2. `agent.active`: status active, within `valid_from` and `valid_until`.
3. `request.valid`: request conforms to the request schema; timestamp within skew limit; nonce unused.
4. `policy.active`: policy version is the active version.
5. `grant.match`: a grant exists for this agent, action, and currency.
6. `grant.window`: request time falls within `valid_window`.
7. `grant.currency`: currency matches.
8. `grant.per_transaction_limit`: amount <= limit.
9. `grant.daily_cap` and `grant.rolling_window_cap`: aggregate within caps.
10. `beneficiary.allowlist`: beneficiary is on the grant and active in the policy.
11. `approval.threshold`: determine required approvals for the amount.
12. `approval.verify`: each supplied approval is signed by a valid approver key, is within its validity window, and satisfies separation of duties.

Only if all twelve return `MATCH` does the decision become ALLOW.

## 8. Linter

Before a policy can be activated, a linter must pass:

- JSON Schema validation.
- Every `agent_id` in a grant exists in `agents`.
- Every `beneficiary_id` in a grant exists in `beneficiaries`.
- Every currency in a grant appears in top-level `currencies`.
- No grant with `per_transaction_limit` greater than `daily_cap`.
- Approval thresholds are satisfiable with the listed groups.
- No `broad: true` grant without `valid_until`.
- No overlapping `valid_window` values that would allow a time-of-day gap to be misread.
- Warnings (not failures) for grants with no `rolling_window_cap`, and for approval thresholds equal to the grant limit.

## 9. Versioning and changes

- Changing any field creates a new `policy_version`.
- Each version is stored in full, never as a diff in the authoritative store.
- Activation requires a signed `policy_activation` receipt referencing the new hash and the previous version hash.
- Old versions remain retrievable so historical receipts can be verified.
