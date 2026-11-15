## Summary

<!-- What does this change do, and why? One or two sentences. -->

## Type of change

- [ ] Specification (`docs/`, `schemas/`, `examples/`)
- [ ] Policy schema change (`schemas/policy.v1.schema.json`)
- [ ] Receipt format change (`docs/RECEIPT_FORMAT.md`)
- [ ] Implementation (gate, executor, verifier)
- [ ] Documentation only (README, comments)
- [ ] Tooling or CI

## Affected documents

<!-- List every document this change touches, and keep them consistent with each other. -->

- [ ] `docs/SPEC.md`
- [ ] `docs/THREAT_MODEL.md`
- [ ] `docs/RECEIPT_FORMAT.md`
- [ ] `docs/POLICY_SCHEMA.md`
- [ ] `schemas/policy.v1.schema.json`
- [ ] `examples/`
- [ ] `README.md`

## Security review

- [ ] Fail-closed behaviour is preserved (SPEC section 6). No path lets an error produce ALLOW.
- [ ] No new way to execute without a consumed ALLOW receipt.
- [ ] No private keys, provider credentials, or real account details in the diff.
- [ ] Any threat added, changed, or made obsolete is reflected in `docs/THREAT_MODEL.md`.

<!-- If any box is unchecked, explain why below. -->

## Owner decisions

<!--
Decisions that belong to the product owner (SPEC section 9, and anything that changes
scope, score policy, or acceptance criteria). Do not resolve these in code. Link the answer
or say "pending".
-->

- Related decisions: <!-- e.g. "SPEC open decision 1 (single-currency MVP)" or "none" -->

## Testing

<!-- What was run, and the result. Write "not run" with the reason for anything not run. -->

- Schema validation (`examples/`): <!-- command and result -->
- Other checks: <!-- command and result, or "not run: reason" -->

## Commit hygiene

- [ ] Every commit has a `Signed-off-by` line (DCO).
- [ ] Commits are signed. If not, say so here and why.

## Reviewer notes

<!-- Where to focus, known gaps, and anything intentionally left out. -->

---

🤖 Generated with [Claude Code](https://claude.com/claude-code)
