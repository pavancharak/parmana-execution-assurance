# Threat Model

Status: Draft v0.1. Each threat needs a test or an explicit accepted-risk entry before MVP.

## Scope

In scope: the PEA gate, its policy store, its receipt log, its signing key, the executor, and the verifier tool.

Out of scope: the payment provider's internals, the customer's identity provider internals, and the physical security of the customer's premises. These are listed as assumptions in section 3.

## 1. Assets

| ID | Asset | Why it matters |
|---|---|---|
| A1 | Ability to move funds | The primary harm |
| A2 | Gate signing key | Forging an ALLOW receipt lets anyone move funds |
| A3 | Policy documents | Changing a policy changes what is authorised |
| A4 | Receipt log | The evidence; tampering destroys trust in all decisions |
| A5 | Approver credentials | Approvals are the human authority |
| A6 | Agent keys | Identify which agent is acting |

## 2. Threats

Ratings use likelihood and impact (L/M/H). Residual risk is after the listed mitigations.

| ID | Threat | Asset | Mitigations | Residual |
|---|---|---|---|---|
| T1 | Compromised agent submits payments outside its grant | A1 | Grant scope checks (amount, currency, beneficiary, window); DENY by default; per-agent daily caps | L/M |
| T2 | Agent forges another agent's identity | A6 | Requests signed with per-agent keys; key registry is policy-controlled; key rotation logged | L/L |
| T3 | Agent bypasses the gate and calls the provider directly | A1 | Provider credentials held only by executor; network policy blocks direct access; gate-issued single-use receipts required by executor | M/H before deployment controls, L/M after |
| T4 | Agent replays an old ALLOW receipt | A1 | Receipts single-use and bound to request nonce; executor records consumed receipt IDs; short receipt expiry | L/L |
| T5 | Agent splits one large payment into many small ones | A1 | Daily and rolling-window caps per beneficiary and per agent; aggregation in rule evaluation | M/M |
| T6 | Attacker steals the gate signing key | A2 | Key held in HSM/KMS, non-exportable; signing requires authenticated session; key-use audit; rotation procedure; receipts carry key ID so rotation is visible | L/H, accepted with monitoring |
| T7 | Attacker forges approval signatures | A5 | Approvals are signed by approver keys; approver keys enrolled through policy; verify before counting | L/M |
| T8 | Single approver approves their own or a colluding payment | A1 | Separation of duties: approvers must differ from the initiating agent's owner and from each other; configurable approver groups | M/M, residual collusion accepted |
| T9 | Policy document modified without authorisation | A3 | Policies immutable and hash-identified; policy changes need signed approval; activation recorded in receipt log; receipts reference the policy hash | L/M |
| T10 | Policy rolled back to a more permissive version | A3 | Monotonic policy version numbers; gate refuses older versions than the highest activated; rollback needs explicit signed approval | L/M |
| T11 | Receipt log altered or truncated | A4 | Hash chain; periodic anchoring of the chain head to an external store the operator does not control; verifier checks continuity | L/M |
| T12 | Receipt log unavailable, so decisions cannot be recorded | A4 | Fail-closed: no receipt, no execution | L/L (availability cost accepted) |
| T13 | Clock manipulation to bypass expiry or date windows | A1 | Clock skew check (60 s); gate uses trusted time source; receipt timestamps signed | L/L |
| T14 | Malformed or oversized request crashes gate and it fails open | A1 | Schema validation before evaluation; any exception is DENY; fuzz testing of request parser | L/L |
| T15 | Policy evaluation error produces ALLOW | A1 | Default-deny evaluation; rules return explicit ALLOW only when they match; errors map to DENY | L/L |
| T16 | Beneficiary account changed between approval and execution | A1 | Receipt binds beneficiary account hash; executor verifies hash before call | L/L |
| T17 | Verifier tool accepts a forged receipt | A4 | Verifier pins expected public key set; rejects unknown key IDs; test vectors in repo | L/L |
| T18 | Insider with gate-admin rights issues an ALLOW directly | A2, A3 | Gate has no manual-allow path; key use requires dual control for signing-key operations; all admin actions logged | M/M |
| T19 | Information leakage in receipts (beneficiary details, amounts) to verifiers | A4 | Receipts contain salted hashes of beneficiary details; full details only with the customer's key | L/L |
| T20 | Supply chain compromise of gate dependencies | A2 | Pinned dependencies, lockfiles, signed releases, reproducible build target | M/H, mitigated partially |

## 3. Assumptions

- The customer's infrastructure hosts the executor and its provider credentials with standard access controls.
- The HSM or KMS vendor's implementation is correct.
- Approvers protect their own credentials. Approver compromise is a T7/T8 concern, not a PEA design fault.
- The payment provider honours the amount and beneficiary in the request it receives.

## 4. Accepted risks (MVP)

- **Collusion between approvers** who are both within the separation-of-duties rules. Mitigation is organisational, not technical.
- **Availability cost** of fail-closed behaviour: a gate outage stops payments. This is intentional.
- **Signing-key compromise (T6)** is not fully preventable. Impact is limited by key rotation and the receipt key ID, and by monitoring for unusual signing volume.

## 5. Required tests before MVP

- One test per threat in section 2 that has a mitigation marked as code-enforced.
- Fuzzing of the request parser and the policy parser.
- Property tests: for any policy and request, the decision is ALLOW only if every rule returns an explicit match.
- Chain tests: tampering with any receipt breaks verification from that point on.
