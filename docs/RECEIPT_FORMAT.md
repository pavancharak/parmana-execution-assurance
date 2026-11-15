# Receipt Format

Status: Draft v0.1. Versioned as `receipt_version: "1"`. Breaking changes require a new major version and a migration note.

## 1. Purpose

A receipt is a signed record of one authorisation decision or one execution event. Receipts let a third party verify that an action was authorised, under which policy, and by whom, without trusting PEA's database.

## 2. Encoding

- Receipts are JSON objects encoded with **JSON Canonicalization Scheme (RFC 8785)** before hashing or signing.
- Hash function: **SHA-256**. Hashes are lowercase hex with a `sha256:` prefix.
- Signature: **Ed25519** over the canonical bytes of the receipt **without** the `signature` field.
- Timestamps: **RFC 3339**, UTC, `Z` suffix, millisecond precision at most.
- Amounts: **integer minor units** (for example, cents) with an ISO 4217 currency code. No floating-point values anywhere in a receipt.
- Identifiers: lowercase UUIDv7 for receipt IDs, to keep them sortable by time.

## 3. Structure

```json
{
  "receipt_version": "1",
  "receipt_id": "0190e5a2-7c1d-7b3a-9f1e-2a4b6c8d0e12",
  "receipt_type": "authorisation",
  "decision": "ALLOW",
  "issued_at": "2026-11-15T10:04:12.381Z",
  "expires_at": "2026-11-15T10:05:12.381Z",
  "gate_id": "pea-gate-prod-1",
  "signing_key_id": "gate-2026-11-a",
  "agent": {
    "agent_id": "agent-ap-bot-01",
    "owner_id": "user-finance-lead-07",
    "key_id": "agent-key-2026-03"
  },
  "request": {
    "request_id": "req-7f3a9c",
    "nonce": "b64:Yx8q2k9mZ0eQ...",
    "action": "payment.create",
    "amount": { "value": 485000, "currency": "GBP" },
    "beneficiary": {
      "beneficiary_id": "ben-acme-supplies",
      "account_hash": "sha256:4f1c0e...",
      "account_salt_ref": "salt-2026-11"
    },
    "requested_at": "2026-11-15T10:04:11.902Z",
    "request_hash": "sha256:9ad21b..."
  },
  "policy": {
    "policy_id": "ap-vendor-payments",
    "policy_version": 14,
    "policy_hash": "sha256:c03e77..."
  },
  "rule_results": [
    { "rule": "agent.active", "result": "MATCH" },
    { "rule": "grant.currency", "result": "MATCH" },
    { "rule": "grant.per_transaction_limit", "result": "MATCH", "detail": { "limit": 500000 } },
    { "rule": "grant.daily_cap", "result": "MATCH", "detail": { "used": 120000, "cap": 2000000 } },
    { "rule": "beneficiary.allowlist", "result": "MATCH" },
    { "rule": "approval.threshold", "result": "MATCH", "detail": { "required": 1 } }
  ],
  "approvals": [
    {
      "approver_id": "user-cfo-02",
      "approval_id": "appr-41aa",
      "signed_at": "2026-11-15T10:03:58.110Z",
      "key_id": "approver-2026-09-cfo02",
      "signature": "ed25519:..."
    }
  ],
  "previous_receipt_hash": "sha256:e82f10...",
  "signature": "ed25519:..."
}
```

## 4. Field definitions

| Field | Required | Description |
|---|---|---|
| `receipt_version` | yes | Format version. Must be `"1"`. |
| `receipt_id` | yes | Unique ID. UUIDv7. |
| `receipt_type` | yes | `authorisation`, `execution`, `policy_activation`, `key_event`. |
| `decision` | yes | `ALLOW` or `DENY`. `DENY` receipts are also signed and logged. |
| `issued_at` / `expires_at` | yes | Issue time and validity window. Authorisation receipts expire within 60 seconds by default. |
| `gate_id` / `signing_key_id` | yes | Which gate instance and signing key issued the receipt. |
| `agent` | yes | Agent identity and owner. |
| `request` | yes | Normalised request. `request_hash` is the hash of the canonical request as received. |
| `policy` | yes | Policy identity and hash used for evaluation. |
| `rule_results` | yes | Every rule evaluated, in order, with `MATCH`, `NO_MATCH`, or `ERROR`. Any `ERROR` forces `DENY`. |
| `approvals` | when required | Approver signatures that were counted. |
| `previous_receipt_hash` | yes | Hash of the previous receipt in the chain. The first receipt uses the genesis hash `sha256:0000...`. |
| `signature` | yes | Ed25519 signature over canonical bytes without this field. |

Execution receipts add:
- `authorisation_receipt_id`: the ALLOW receipt consumed.
- `provider_reference`: the provider's transaction ID.
- `execution_status`: `SUBMITTED`, `SETTLED`, `FAILED`, `REJECTED_BY_PROVIDER`.

## 5. Chain rules

- Every receipt in one log stream has `previous_receipt_hash` equal to the SHA-256 of the canonical bytes of the previous receipt, including its signature.
- Gaps, duplicates, or forks are verification failures.
- Chain heads are anchored externally at a fixed interval (default hourly). The anchor is a signed statement of the head hash and sequence number.

## 6. Verification procedure

A verifier, given a receipt and its trusted key set, must:

1. Confirm `receipt_version` is supported.
2. Canonicalise the receipt without `signature` and verify the Ed25519 signature against the key identified by `signing_key_id`. Reject unknown key IDs.
3. Check `issued_at` <= now <= `expires_at` for authorisation receipts when verifying at the time of use. For historical verification, skip this check and record that it was skipped.
4. Confirm the `policy_hash` matches the supplied policy document, if one is provided.
5. For each approval, verify the approver signature against an approver key that was valid at `signed_at`.
6. If the receipt is part of a chain, verify `previous_receipt_hash` against the supplied predecessor.
7. Confirm `decision` is `ALLOW` only if every `rule_results` entry is `MATCH` and approvals meet the policy requirement.

Failure at any step means the receipt is invalid. The verifier reports the first failing step.

## 7. Privacy

- Beneficiary account numbers never appear in plain text. Only a salted hash appears. Salts are held by the customer.
- Receipts do not contain provider credentials, personal data beyond identifiers, or free-text memo fields in MVP.

## 8. Test vectors

The repo will include canonical test vectors: a valid ALLOW chain, a DENY receipt, a receipt with a tampered amount, a receipt with an unknown key ID, and a broken chain. Verifiers in any language must pass all of them before claiming compatibility.
