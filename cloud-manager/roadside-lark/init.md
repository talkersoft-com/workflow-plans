# Cloud Manager controller — systemd-creds secret injection

## What this workflow does
Moves the controller's secrets (`VAULT_TOKEN`, `DBPASSWD`, `RABBITMQ_PASSWORD`) out of
plaintext unit/env files into systemd encrypted credentials. An idempotent `cm-seed-secrets`
script auto-generates strong passwords (or reuses what's in Vault), applies them to
Postgres/RabbitMQ, stores them in Vault, and encrypts them into `.cred` files; the API gets a
one-line `AddKeyPerFile` change to read them. When done, no secret is in any unit/env file,
`systemctl cat` is clean, and an end-to-end VM provision still succeeds. Debian packaging is
out of scope.

## Read before starting
- `deck.md` — deck + repos in scope; pre-written hv MCP calls
- `Execution/Exec.md` — full task list and execution instructions
- `../../common/toolkit/hive-deck.md` — hv MCP call patterns
- `PLAN.md` — objective, seeder design, injection model, phases

## Constraints
- The seeder is idempotent: reuse-from-Vault if present, else generate; **never** regenerate
  the Vault token, **never** clobber an existing stored password.
- No secret in any `Environment=` line or plaintext file once cutover is complete; canonical key
  names are fixed (`VAULT_TOKEN`, `DBPASSWD`, `RABBITMQ_PASSWORD`).
- RabbitMQ password rotation must reach ALL three consumers (API, vorch, porch).
- Back up BEFORE live cutover: Vault snapshot (`snapshot-vault.py`) + `pg_dump clouddb`.
- Every task self-contained and idempotent — execution runs headless against ubuntu-server.
- Out of scope: Debian packaging (separate later effort).
