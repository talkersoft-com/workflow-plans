# Plan: Secret-binding credential injection must work on Ubuntu 22.04 AND 24.04

> TDD task. The acceptance test already exists and is the **sole** definition of done.
> There is intentionally **no "Open questions" section** — execute end-to-end without
> pausing for operator input; resolve any ambiguity from the codebase and the test.

## Objective

Make `cloud-manager-api/tests/test_secret_injection.py` exit **0**. That test asserts that a
VM provisioned from the `database-ready-web-server-jammy` (22.04) **and** the
`database-ready-web-server-noble` (24.04) blueprints — each with the `DBPASSWD` secret binding
pointed at a Postgres VM — boots with the DB credential **injected**, and that the deployed
sample app, reading the password **only** from the injected systemd credential, connects to
Postgres and serves `Database connection: SUCCESS` through haproxy on 443. Today the 22.04 case
is **verified broken** and 24.04 is unverified; when this work is done **both cases pass**.

## Background

- **Root cause (verified — do NOT re-investigate).** The on-VM seal step
  (`vorch-lib` `injectSecrets` → generated `seal-credentials.sh`) runs `systemd-creds encrypt`,
  which only ships with **systemd ≥ 250**. Ubuntu 22.04 has **systemd 249** → `systemd-creds`
  is **MISSING** → the seal fails **silently** → `/etc/credstore.encrypted` stays empty →
  nothing is injected. This is despite the API resolving the binding correctly (a `vsb_…` row
  with `resolvedVaultPath` is created) and the base64 secret + `seal-credentials.sh` being present
  in `/run/cm-seed` on the VM. 24.04 (systemd 255) has `systemd-creds`; the noble case verifies
  whether that path is fully correct.
- **Two defects:** (1) the design assumes systemd ≥ 250; (2) on failure the plaintext password is
  left in `/run/cm-seed` (the shred never runs) and nothing is surfaced.
- **The cert path is unaffected** — it rides the `x-secret-path` extra-var mechanism, not
  `systemd-creds`, which is why haproxy/TLS already works.

## Design

Make injection **version-aware** so the guest can load the credential on both systemd 249 and 255,
and the consuming unit reads it the same way (`$CREDENTIALS_DIRECTORY/DBPASSWD`). The recommended
shape (the implementer may choose another **iff** the test passes and the constraints hold):

- **systemd ≥ 250 (noble):** keep `systemd-creds encrypt` → `/etc/credstore.encrypted/DBPASSWD`;
  consume with `LoadCredentialEncrypted=DBPASSWD`.
- **systemd < 250 (jammy):** fall back to a **root-only (0600 root)** plaintext credstore file
  (e.g. `/etc/credstore/DBPASSWD`); consume with `LoadCredential=DBPASSWD:/etc/credstore/DBPASSWD`
  (`LoadCredential` exists since systemd 247).
- The generated seal script detects the guest's systemd version and picks the path.
- **Always shred** `/run/cm-seed/*` (success or failure); make a seal failure **loud**
  (non-zero / surfaced on the run), never silent.

### Files in scope
- `vorch-lib/provision/provision.go` — `injectSecrets` and the generated `seal-credentials.sh`:
  version-aware seal, guaranteed shred, loud failure.
- `cloud-manager-api/tests/hello-db-web/hello-db-web.service` and `deploy.sh` — the consuming unit
  picks `LoadCredential` vs `LoadCredentialEncrypted` by the VM's systemd version (the sample stands
  in for how a real app consumes the credential).
- `cloud-manager-api/tests/hello-db-web/server.js` — keep credential-only (reads
  `$CREDENTIALS_DIRECTORY/DBPASSWD`, no env-password path).

### Constraints (keep the test honest — do NOT violate)
- Do **not** edit the test's assertions/gate (`test_secret_injection.py`) to force a pass.
- The app must obtain the DB password **only** from the binding-injected systemd credential — no
  hardcoding, no `PGPASSWORD` env, no extra Vault fetch in the app/playbook, nothing baked into the image.
- Secret values never logged, never in run output, never world-readable; the 22.04 fallback file is `0600 root`.
- `vorch-lib` + consumption fix only — **no API / web / DB-schema changes** unless strictly required.
- VMs are never Vault clients (the controller injects).

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000 | Setup | `hv_status` + `hv_next`; record the branch |
| 0001 | vorch-lib version-aware seal | `injectSecrets`/`seal-credentials.sh`: detect guest systemd version; ≥250 → `systemd-creds encrypt` → `/etc/credstore.encrypted/<name>`; <250 → root-only `0600` file at `/etc/credstore/<name>`; **always** shred `/run/cm-seed`; fail loud on seal error. Unit/golden test of the generated user-data for both branches. |
| 0002 | sample consumption per-OS | `tests/hello-db-web`: `deploy.sh` + `hello-db-web.service` select `LoadCredentialEncrypted` (≥250) vs `LoadCredential` (<250) by the VM's systemd version; `server.js` stays credential-only. |
| 0003 | E2E verify + deploy | build `vorch`; **manually redeploy `vorch-service`** on the host (record the exact command); run `cloud-manager-api/tests/test_secret_injection.py` on ubuntu-server; **both** cases must print PASS and the script must exit 0 (paste the final RESULT block); confirm **no plaintext** remains in `/run/cm-seed` on either VM after boot; write Results + LESSONS. |

## Acceptance criteria (exec verifies — never asks)
- `tests/test_secret_injection.py` exits 0 with **both** the jammy (22.04) and noble (24.04) cases PASS.
- Run on ubuntu-server; token via the systemd-sealed `vault-token` credential.
- No plaintext credential left in `/run/cm-seed` on either VM after boot.
- No API/web/DB-schema changes; the haproxy/wildcard-cert path still works.

## Out of scope
- The haproxy / wildcard-cert path (already works).
- Changing the DB or its blueprint.
- Any mechanism that lets the app read the password without going through the binding.

## Prerequisites (already live in the CM instance the agent uses)
- Published blueprints `database-ready-web-server-jammy` and `database-ready-web-server-noble`,
  each with the `haproxy-tls-proxy` playbook + the `DBPASSWD` (`pg-consumer-binding`) secret binding.
- The `wildcard-talkersoft-com` Vault secret; the `vault-token` systemd credential on the host
  (re-seed with `cloud-manager-api/tests/seed-vault-token.sh` if missing).
