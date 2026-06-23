# Cloud Manager controller — Debian packaging + systemd-creds secret injection

## What this workflow does
Packages the Cloud Manager controller stack (cloud-manager-api, vorch_service, porch) and
the web SPA as reproducible Debian packages that install fresh AND adopt the existing
controller, injecting all secrets through systemd encrypted credentials instead of plaintext
unit/env files, and installing a timer that keeps the (already-minted) 1-year periodic Vault
token renewed. When done, `apt install cloud-manager` stands up or upgrades a controller with
no secret ever written to a unit file, and an end-to-end VM provision succeeds.

## Read before starting
- `deck.md` — deck + repos in scope; pre-written hv MCP calls
- `Execution/Exec.md` — full task list and execution instructions
- `../../common/toolkit/hive-deck.md` — hv MCP call patterns
- `PLAN.md` — objective, design (secret-loading contract, Layer 0/1 bootstrapping), phases

## Constraints
- **NEVER** run `vault operator init` on an already-initialized Vault; **never** drop/recreate
  `clouddb`; **never** overwrite an existing token `.cred`. postinst is idempotent
  detect-and-adopt only.
- No secret in any `Environment=` line or plaintext file once migration is complete; canonical
  key names are fixed (`VAULT_TOKEN`, `DBPASSWD`, `RABBITMQ_PASSWORD`).
- RabbitMQ password rotation must update ALL three consumers (API, vorch, porch).
- Every step idempotent and re-runnable — execution runs headless against ubuntu-server after a
  teardown/recreate bootstrap.
- Verify on the live controller only AFTER a Vault snapshot + `pg_dump clouddb` backup.
