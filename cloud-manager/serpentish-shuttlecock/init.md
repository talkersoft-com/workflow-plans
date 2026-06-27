# Secret-binding injection on Ubuntu 22.04 AND 24.04 (TDD)

## What this workflow does
Fixes secret-binding **credential injection** so a provisioned VM actually receives its bound
secret as a systemd credential on **both** Ubuntu 22.04 (jammy) and 24.04 (noble). Today the seal
step uses `systemd-creds encrypt`, which is missing on jammy (systemd 249), so injection fails
silently and the credstore is empty. The fix makes the seal **version-aware** and the consuming
unit read the credential the same way on both releases. Done = the existing acceptance test passes.

## The acceptance test IS the spec
`cloud-manager-api/tests/test_secret_injection.py` — idempotent, MCP-driven, runs two OS cases
(jammy + noble) that must **both** pass. Run it on ubuntu-server:
```
cloud-manager-api/tests/test_secret_injection.py
```
Success = exit 0 with both cases PASS. Do not edit its assertions to force a pass.

## Read before starting
- `PLAN.md` — root cause (verified), the version-aware design, phases, acceptance criteria
- `deck.md` — repos in scope (`vorch-lib`, `cloud-manager-api`); the mandatory vorch redeploy + test run
- `Execution/Exec.md` — full task list and execution instructions
- `../../common/toolkit/hive-deck.md` — hv MCP call patterns

## Constraints (all decided — do not defer to the operator)
- The app must get the DB password **only** from the binding-injected systemd credential — no
  hardcoding, no `PGPASSWORD` env, no extra Vault fetch, nothing baked into the image.
- Secret values never logged / never world-readable; the 22.04 fallback file is `0600 root`.
- Always shred `/run/cm-seed`; a seal failure must be **loud**, never silent.
- `vorch-lib` + the sample consuming unit only — no API/web/DB-schema changes unless strictly required.
- VMs are never Vault clients (the controller injects).
- Verify for real: build vorch → redeploy vorch-service on the host (record the command) → run the
  test on ubuntu-server → both PASS → confirm no plaintext left in `/run/cm-seed`.

## Out of scope
- The haproxy / wildcard-cert path (already works); changing the DB or its blueprint.
