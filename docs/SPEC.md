# Parmana Execution Assurance: Product Specification

Status: Draft v0.1 for review. Not implemented.

## 1. Purpose

Parmana Execution Assurance (PEA) makes every financial action prove it was authorised before it executes. It sits in the execution path of a payment or transfer and does one thing: it allows an action only when an explicit, signed authority exists for that exact action, and it produces signed, verifiable evidence of the decision.

PEA is a control and an evidence producer. It is not a compliance attestation, and it does not replace the organisation's own approval process, audit, or legal review.

## 2. Target user and MVP scope

- **Buyer:** finance operations and internal audit teams at mid-size organisations that use software agents, scripts, or automated workflows to initiate payments.
- **MVP rail:** outbound vendor payments through one payment provider's sandbox API. No other rails in MVP.
- **MVP actions:** `payment.create` only. Refunds, payroll, and FX are out of scope.
- **MVP capabilities:**
  1. Register agents with a public key and an owner.
  2. Load a versioned policy (see `docs/POLICY_SCHEMA.md`).
  3. Evaluate a payment request against the policy and approval records.
  4. Allow or deny. Denied requests never reach the provider.
  5. Issue a signed receipt for every decision (see `docs/RECEIPT_FORMAT.md`).
  6. Verify a receipt offline with a published public key and the policy hash.

## 3. Non-goals (MVP)

- Detecting fraud, sanctions screening, or KYC/AML checks.
- Deciding whether a payment is commercially sensible.
- Managing the payment provider's own security or settlement.
- Acting as a general-purpose policy engine for non-financial actions.
- Hosting or custodying funds.

## 4. Actors

| Actor | Trust level | Role |
|---|---|---|
| Agent | Untrusted | Requests actions. May be compromised or mistaken. |
| Approver (human) | Trusted for their own approvals, not for others' | Signs approvals for specific payments. |
| Policy author | Trusted, audited | Publishes policy versions. |
| PEA gate | Trusted computing base | Evaluates requests and signs receipts. |
| Executor | Trusted, minimal | Calls the provider only with a valid ALLOW receipt. |
| Verifier | Untrusted | Any party checking a receipt. |

## 5. Execution flow

1. **Identify.** The agent submits a request signed with its registered key. The request includes an agent ID, a nonce, and a timestamp.
2. **Authorise scope.** The gate looks up the agent's grants in the active policy version and confirms the agent is registered, active, and not revoked.
3. **Validate.** The gate checks the request against explicit rules: currency, per-transaction limit, daily cap, beneficiary allowlist, and time window.
4. **Verify approvals.** If the amount requires approval under the policy, the gate verifies the required approver signatures. Approvers must be distinct from each other and from the initiating agent's owner (separation of duties).
5. **Decide.** The gate produces ALLOW or DENY, with the reasons for each rule result.
6. **Record.** The gate appends the decision to the hash-chained receipt log and signs it.
7. **Execute.** The executor calls the provider only if it holds an ALLOW receipt that is signed, unexpired, and not yet used. The executor marks the receipt as consumed.
8. **Evidence.** The execution result is recorded in a follow-up receipt that references the authorisation receipt.

## 6. Fail-closed rules

These are mandatory and have no configuration override:

- Missing, unparseable, or unsigned policy: DENY.
- Unknown, revoked, or expired agent key: DENY.
- Any rule evaluation error or exception: DENY.
- Clock skew beyond 60 seconds between the request and the gate: DENY.
- Receipt log cannot be written: DENY. No receipt, no execution.
- Reused nonce or already-consumed receipt: DENY.
- Executor receives no valid ALLOW receipt: no call to the provider.
- No "dry-run allow" or "log-only" mode in production builds. Test modes must be compiled out or require an explicit non-production environment flag that the gate refuses in production.

## 7. Requirements

**Functional**
- F1. Every payment request produces exactly one authorisation receipt, including denials.
- F2. Policy versions are immutable and identified by hash.
- F3. Receipts verify offline using only the receipt, the public key, and the policy document.
- F4. The executor cannot execute a payment without a consumed ALLOW receipt.

**Security**
- S1. Signing keys are held in an HSM or KMS. The gate never exports the private key.
- S2. Receipts are signed with Ed25519 over a canonical JSON encoding.
- S3. Receipt log is append-only and hash-chained. Each receipt references the hash of the previous one.
- S4. Gate-to-executor communication is authenticated.

**Operational**
- O1. Gate decision latency under 200 ms at p99 for the MVP rail, excluding provider calls.
- O2. Receipt log retention is configurable, with a default of 7 years.
- O3. Every policy change and key rotation is itself recorded in the receipt log.

## 8. Success criteria for MVP

- All fail-closed rules in section 6 have automated tests that run in CI.
- A verifier can validate a receipt chain end to end with the published tool and no access to PEA internals.
- Adversarial test suite covers the threats in `docs/THREAT_MODEL.md` (each threat has at least one test or an explicit "accepted risk" entry).
- Pilot run against the payment provider's sandbox with no unauthorised executions observed.

## 9. Open decisions for the owner

1. Single-currency MVP or multi-currency from the start?
2. Who holds the approver registry: PEA, or the customer's identity provider?
3. Is the executor deployed inside the customer's network or as a PEA-hosted service? This changes the trust boundary.
4. Required approver count and thresholds: set by PEA defaults, or fully customer-defined?
5. Licensing model for the verifier tool (open source to enable independent verification, or closed).
