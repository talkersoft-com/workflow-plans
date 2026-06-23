# Plan: systemd-creds secret injection for the Cloud Manager controller

## Objective

Make the controller's services — `cloud-manager-api`, `vorch_service`, `porch` — read
all their secrets (`VAULT_TOKEN`, `DBPASSWD`, `RABBITMQ_PASSWORD`) from **systemd
encrypted credentials** instead of plaintext unit/env files. An idempotent **seed
script** is the single authoritative seeder: it auto-generates strong passwords (or
reuses what's already in Vault), applies them to Postgres/RabbitMQ, stores them in Vault,
and encrypts them into `.cred` files. When done, no secret exists in any unit file or
plaintext env file, `systemctl cat` is clean, and an end-to-end VM provision still works.
**Debian packaging is explicitly out of scope** (separate, later effort).

## Background

- **Live state:** the Vault token outage is already fixed — a 1-year **periodic**
  `cloudmanager` token runs (after raising `auth/token` `max_lease_ttl` to `8760h`),
  verified by a successful provision. This plan hardens *how* secrets are stored, not the token itself.
- **Plaintext today:** the `cloud-manager-api` unit carries `Environment=DBPASSWD=P@ssw0rd!`
  and `RABBITMQ_PASSWORD=P@ssw0rd!` (readable via `systemctl cat`); the Vault token lived in
  three places (`api.env`, `config-server.yaml`, `porch.service`).
- **Platform:** Ubuntu 24.04, systemd 255, `systemd-creds` present (TPM2 partial) →
  `LoadCredentialEncrypted` available (host/TPM-bound, tmpfs, per-service, 0400).
- **Auth model (confirmed in code):** the Go vault client already supports `VAULT_TOKEN_FILE`;
  the .NET `VaultClient` reads env only (needs the one change below).
- **Proven pattern to reuse:** `pg-14-jammy` / `rabbitmq-jammy` playbooks already do
  "check Vault → reuse if present, else generate 32-char random → store in Vault." The seed
  script applies that same pattern to the controller's *own* Postgres/RabbitMQ.
- **Purpose:** primarily to exercise the agentic **hive-deck → headless** workflow end to end;
  execution is delegated to the agent on the server via `headless-run.py exec`.

## Design

### The seeder — `cm-seed-secrets` (authoritative, idempotent)
A single script run on the controller. Per secret:
1. **value** — reuse-from-Vault if present, else generate a strong 32-char random password
   (auto-generated; never human-chosen).
2. **apply** — Postgres `ALTER ROLE cloudmanager`, RabbitMQ `rabbitmqctl change_password`.
3. **store** — write to Vault (source of truth): `cloudmanager/controller/postgres`,
   `cloudmanager/controller/rabbitmq`.
4. **inject** — `systemd-creds encrypt` → `/etc/cloud-manager/creds/<name>.cred`.

`VAULT_TOKEN` is already minted — the seeder only **encrypts the existing value** to
`vault-token.cred`, never regenerates it. Re-running reuses stored values and never clobbers.

### Injection — fixed names, files not env vars
- Units gain `LoadCredentialEncrypted=<name>:/etc/cloud-manager/creds/<name>.cred` →
  systemd decrypts to `$CREDENTIALS_DIRECTORY/<name>` (tmpfs, 0400, per-service).
- **vorch/porch (Go):** `Environment=VAULT_TOKEN_FILE=%d/vault-token` — already supported, no code change.
- **API (.NET):** ONE change — `AddKeyPerFile($CREDENTIALS_DIRECTORY)` layered under env
  (`env → cred`), so the cred file named `DBPASSWD` becomes config key `DBPASSWD`. Existing API
  code keeps working; it just sources values from decrypted files. Backward-compatible.
- Remove every plaintext `Environment=<secret>` line from the units.

### Three-touch-point note (the only fiddly part)
RabbitMQ creds are consumed by **API + vorch + porch**. API reads its cred via `AddKeyPerFile`;
**vorch/porch read RabbitMQ creds from `config-server.yaml` today**, so the seeder/cutover must
update those too (or hand vorch/porch a cred). Must be handled explicitly, not assumed.

### Cutover (live controller) — reversible
**Back up first:** Vault snapshot (`snapshot-vault.py`) + `pg_dump clouddb`. Then run the seeder,
redeploy the API, restart api/vorch/porch, strip plaintext.

## Implementation phases

Each phase = one TASK the headless agent executes; every task self-contained and idempotent.

| Phase | Title | Description | Test |
|-------|-------|-------------|------|
| 0000 | Setup | branch via hv | clean status |
| 0001 | `cm-seed-secrets` script | reuse-or-generate → apply → store in Vault → `systemd-creds encrypt` → `.cred` | dry-run produces creds; second run reuses (idempotent) |
| 0002 | API `AddKeyPerFile` change | layered `env → cred` config provider over `$CREDENTIALS_DIRECTORY` | API resolves a secret from a cred file |
| 0003 | systemd unit wiring | `LoadCredentialEncrypted=` on api/vorch/porch + `VAULT_TOKEN_FILE`; RabbitMQ cred for vorch/porch | creds appear at `$CREDENTIALS_DIRECTORY` (0400); units carry no secret |
| 0004 | Live cutover | backup (Vault snapshot + `pg_dump`) → run seeder → redeploy API → restart → strip plaintext | services active; `systemctl cat` shows no secrets |
| 0005 | Verify | end-to-end VM provision | SSH key written to Vault, IP assigned |

## Open questions

*(All major decisions locked: home repo = `vm-infra/secret-management`; auto-generated passwords;
backup-first; controller-only scope. None blocking.)*
- Confirm whether vorch/porch should read the RabbitMQ password from a **systemd cred** (preferred,
  consistent) or keep reading `config-server.yaml` with the seeder updating that file — decide in 0003.
