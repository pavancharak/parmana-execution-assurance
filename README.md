# Parmana Execution Assurance

Make every financial action prove it was authorised before execution.

**Status: specification draft (v0.1). No implementation yet.**

## What it does

PEA sits in the execution path of a payment and:

1. **Identifies the agent** and its declared authority.
2. **Validates the request** against explicit business rules.
3. **Verifies amount, beneficiary, and approval requirements** against a versioned policy.
4. **Blocks unauthorised execution.** Any request it cannot prove was authorised is denied (fail closed), and the provider is never called.
5. **Produces cryptographically verifiable evidence.** Every decision yields an Ed25519-signed, hash-chained receipt that a third party can verify offline.

PEA is a control that produces evidence. It is not a compliance attestation, and it does not replace your organisation's approval process or audit.

## MVP scope

- One payment rail (outbound vendor payments via one provider's sandbox API).
- One action: `payment.create`.
- Denials are recorded and signed, the same as approvals.

Non-goals for MVP: fraud or sanctions screening, commercial judgement of payments, fund custody, and non-financial actions.

## Documents

- [`docs/SPEC.md`](docs/SPEC.md): scope, actors, execution flow, fail-closed rules, requirements, success criteria, open decisions.
- [`docs/THREAT_MODEL.md`](docs/THREAT_MODEL.md): assets, threats, mitigations, residual risk, accepted risks.
- [`docs/RECEIPT_FORMAT.md`](docs/RECEIPT_FORMAT.md): receipt structure, canonical encoding, chain rules, verification procedure.
- [`docs/POLICY_SCHEMA.md`](docs/POLICY_SCHEMA.md): policy structure, approval rules, grants, evaluation order, linter.
- [`schemas/policy.v1.schema.json`](schemas/policy.v1.schema.json): machine-readable policy schema (JSON Schema 2020-12).

## Design principles

- **Fail closed.** Missing policy, unknown keys, rule errors, clock skew, or an unwritable receipt log all result in DENY. There is no log-only or dry-run allow in production.
- **Default deny.** An action is allowed only when a grant explicitly matches and every rule passes.
- **Immutable policy.** Policy versions are never edited, are identified by hash, and only move forward.
- **Offline verification.** Receipts verify with the public key set and the policy document, without access to PEA internals.

## Open decisions

Listed in section 9 of [`docs/SPEC.md`](docs/SPEC.md). These need an owner decision before implementation starts.

## Contributing

Work happens on `claude/` branches. Commits carry a `Signed-off-by` line (DCO).
