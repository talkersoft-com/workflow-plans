# Plan: Vault-backed database connections — wire the resolver, verify provisioning gaps

## Objective

Make vault-backed database config entries actually work in the database-toolkit MCP, and close out two adjacent findings from the 2026-06-10 pg-test connectivity session with evidence-based investigations rather than speculative fixes. End state: `db_test_connection` (and every other db tool) succeeds against the `pg-test` entry whose credentials live in Vault, and we have written findings on (a) whether postgres VM provisioning should configure network access and (b) who writes/reads the double-encoded per-VM SSH secret.

## Background

On 2026-06-10 an end-to-end connectivity test against the pg-test VM (`vm_SMAZ3C2YVZ`, 10.0.150.61) surfaced three issues:

1. **Verified bug** — `tools/database-toolkit/tools/shared/vault-resolver.ts` (`resolveVaultEntry` / `resolveVaultEntries`) exists in source and dist but is imported by nothing. Config entries using `vaultPath` (e.g. `pg-test` → KV v2 `cloudmanager/vm/instances/vm_SMAZ3C2YVZ/postgres`) are returned without host/port/username/rootPassword, so `db_test_connection` fails with "Database connection requires host, port, username, and rootPassword" even when Vault is reachable and the Keychain token (`vault-admin-token`) is valid. Reproduced live; confirmed by grep that no call sites exist.
2. **Observed once, cause unknown** — the pg-test VM came up with PostgreSQL 14 bound to 127.0.0.1 and a localhost-only pg_hba, unreachable even from the KVM host. Fixed manually on 2026-06-10 (`listen_addresses='*'`, scram entries for 10.0.150.0/24 and 10.0.144.140/32). We do NOT know whether a playbook owns postgres setup on that VM; do not codify anything until that is established.
3. **Observed, writer/readers unmapped** — Vault secret `cloudmanager/vm/instances/vm_SMAZ3C2YVZ/ssh` has a single field `ssh_key` whose value is itself a JSON string (`{"private_key":"...","created_at":"..."}` with escaped newlines), forcing consumers to double-parse. Changing the shape without mapping all readers (e.g. `cloud-manager-cli` `internal/vaultclient` used by `cm ssh`) risks breaking working consumers.

## Design

### Phase 1 — wire the Vault resolver (tools/database-toolkit)

Integrate `resolveVaultEntries` into the config-loading path in `tools/shared/database-config-loader.ts`:

- `loadDatabaseConfigWithFallback()` and `loadTestDatabaseConfigWithFallback()` resolve vault-backed entries after merging project + fallback configs, before returning.
- Merge semantics are already implemented in the resolver and must be preserved: Vault supplies `host`, `port`, `username` (from `user`), `rootPassword` (from `password`); **explicit entry fields win** (e.g. `host: localhost` routes through an SSH tunnel instead of the VM's real IP).
- Token source order: `CM_VAULT_TOKEN` env, then macOS Keychain item `vault-admin-token` — as already implemented in the resolver; do not change. `VAULT_ADDR` env overrides the default `https://ubuntu-server.talkersoft.com:8200`.
- A Vault failure on one entry must not break other entries (the resolver already isolates per-entry failures) and must never log secret material.
- Non-vault entries (`clouddb`, `appdb`, `clouddb_local`) must be returned byte-identical to today.
- Rebuild dist and restart the MCP server per the rebuild-mcp-artifacts rules.

Acceptance: with an SSH tunnel `localhost:5432 → 10.0.150.61:5432` and the `host: localhost` override present in `~/.eng-data-tools/config/database.json`, `db_test_connection` with `dbKeyIdentifier: "pg-test"` succeeds; `db_list_databases` / `db_list_tables` / `db_execute_sql_query` work against it; all non-vault configs behave exactly as before.

### Phase 2 — investigation: postgres VM network provisioning

Determine how PostgreSQL got onto the pg-test VM: search cloud-manager playbooks/roles (via cloud-manager-mcp playbook tree and the API database) for the postgres install. Deliverable is a findings document:

- If a playbook/role owns postgres setup: recommend (and only then implement) adding `listen_addresses` + pg_hba configuration scoped to the VM network and KVM host source IP, scram-sha-256 only. The change must be idempotent and affect newly provisioned VMs only.
- If postgres was hand-installed: document that; recommend whether a postgres role should exist. Do not invent a new playbook in this plan.

### Phase 3 — investigation: double-encoded per-VM SSH secret

Map the full producer/consumer graph of `cloudmanager/vm/instances/<vm-id>/ssh`:

- Writer: expected in vorch-lib's provision/vault path — **note the deck convention forbids vorch changes**, so if the writer is vorch-lib the recommendation must flag that explicitly rather than change it.
- Readers: `cloud-manager-cli` `internal/vaultclient` (`cm ssh`), porch, any scripts under `cloud-manager-api/scripts/`.
- Deliverable is a written recommendation, no code change: either (a) normalize the secret shape in one coordinated change covering the writer, a migration for existing secrets, and every reader, or (b) document the current double-encoded shape as the contract. Include the blast radius for each option.

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000  | Setup | hv_status + hv_init/hv_next |
| 0001  | Wire vault-resolver into database-toolkit config loaders | Integrate `resolveVaultEntries` into `loadDatabaseConfigWithFallback`/`loadTestDatabaseConfigWithFallback`, rebuild dist, verify pg-test end-to-end and non-vault configs unchanged |
| 0002  | Investigate postgres VM provisioning gap | Find what installed postgres on pg-test; findings doc + conditional, scoped playbook change only if a playbook owns postgres setup |
| 0003  | Investigate double-encoded SSH secret | Map writer + all readers of `vm/instances/<id>/ssh`; written recommendation only, no code change |

## Open questions

- Should the pg_hba scope for provisioned postgres VMs be the VM subnet only, or also include the KVM host's source IP (observed: tunnel traffic arrives from 10.0.144.140)? Phase 2's findings should propose one.
- If Phase 3 recommends normalizing the secret shape, does that work belong to a separate plan given the vorch no-change convention? (Default assumption: yes, separate plan.)
