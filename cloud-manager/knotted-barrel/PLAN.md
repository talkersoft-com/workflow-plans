# Plan: `manager` database provider for the database-toolkit MCP

## Objective

Let the database-toolkit MCP connect to cloud-manager VMs with **no credentials in local config** —
resolved from Vault and reached over an **SSH tunnel** — by adding a `manager` provider alongside
the existing literal-creds `standard` provider. This closes the integration gap found in the live
test: `cloud_db_config_add` already registers a Vault/SSH entry, but the toolkit can't consume it,
so `db_test_connection` failed and we fell back to raw `psql`. When done, an operator runs
`cloud_db_config_add` for a VM and then every `db_*` tool (`db_test_connection`,
`db_execute_sql_query`, `db_list_*`, …) works against it with creds staying in Vault. In the
`database-toolkit` repo only; no api/web changes. This is workflow **#6** of the security track.

## Background

- **The gap (verified live):** `cloud_db_config_add` (cloud-manager MCP) writes
  `{provider, vmId, postgresVersion, tunnel:"ssh"}` to `~/.hv/toolkit/db/config/database.json` and
  promises all `db_*` tools can use it. But `db_test_connection` reads a *different* path
  (`<workspace>/.eng-data-tools/config/database.json` + MCP-tools fallback) and requires **literal**
  `host/port/username/rootPassword` — it has no provider concept. Result:
  `requires host, port, username, and rootPassword`.
- **Where the data is:** Postgres creds in Vault at `cloudmanager/data/vm/instances/{vmId}/postgres`
  (host, port, db, user, password); per-VM SSH key at `cloudmanager/data/vm/instances/{vmId}/ssh`
  (`ssh_key` JSON → `private_key`); SSH user `cloudmanager`; Vault `https://ubuntu-server.talkersoft.com:8200`;
  token from keychain (`security find-generic-password -a vault-admin-token -w`).
- **Why a tunnel:** with the #5 firewall, only SSH (22) is open to off-subnet clients. Tunneling to
  the VM's own SSH → `127.0.0.1:5432` reaches Postgres via the default localhost `pg_hba` rule, and
  works **with or without** the firewall. (Independent of #5.)

## Design

### 1. Provider abstraction (config)
Add a `provider` field to each `database.json` entry:
- **`standard`** (default — today's behavior): literal `host/port/username/rootPassword`. Unchanged, fully backward compatible.
- **`manager`** (new): `{ vmId, postgresVersion }`, **no creds**. Triggers Vault/SSH resolution.

### 2. Unify the config path
The toolkit also reads `~/.hv/toolkit/db/config/database.json` (where `cloud_db_config_add` writes),
in addition to the existing project/MCP-tools locations — so a registered DB shows up in
`db_get_config_identifiers` and is usable by every `db_*` tool. Reconcile any field-name mismatch in
the toolkit's reader (don't change `cloud_db_config_add`).

### 3. `manager` resolution (connection layer)
When an entry is `provider:manager`, at connect time:
1. read DB creds from Vault `cloudmanager/data/vm/instances/{vmId}/postgres` (host, port, db, user, password) using the keychain token + Vault addr;
2. read the SSH private key from `cloudmanager/data/vm/instances/{vmId}/ssh`;
3. open an SSH tunnel — `ssh -i <key> cloudmanager@<host> -L <localPort>:127.0.0.1:<port>` (to the VM's own SSH on 22, forwarding to Postgres on the VM's localhost);
4. connect Postgres to `127.0.0.1:<localPort>` with the resolved user/db/password;
5. tear the tunnel down when the op completes (ephemeral per-op; optional simple cache).

### 4. All `db_*` tools transparent
Resolution lives in the **connection layer**, so `db_test_connection`, `db_execute_sql_query`,
`db_list_databases/schemas/tables`, `db_get_table_schema`, etc. work unchanged against a `manager`
DB. Secret **values** never appear in output (only status / row counts / schema, per existing rules).

### End state
```
operator: cloud_db_config_add (vmId, name, pgVersion)   →  ~/.hv/.../database.json {provider:manager}
operator: db_test_connection (dbKeyIdentifier=name)     →  Vault creds + SSH tunnel → ✅ connected
                                                             (no creds ever in local config)
```

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000 | Setup | hv_status + hv_init/hv_next; record branch |
| 0001 | Config: provider field + shared path | add `provider` (`standard` default / `manager`); read `~/.hv/toolkit/db/config/database.json`; `standard` unchanged; `db_get_config_identifiers` lists manager entries |
| 0002 | `manager` resolution: Vault + SSH tunnel | fetch DB creds + SSH key from Vault; open/close SSH tunnel to VM localhost:5432; wire into the connection layer so all `db_*` tools use it; no value logging |
| 0003 | E2E + build | provision (or reuse) a pg VM → `cloud_db_config_add` → `db_test_connection` + `db_execute_sql_query` succeed via the manager provider (Vault + tunnel); a `standard` config still works; tunnels cleaned up; build; local-rebuild + `/mcp` note; write Results + LESSONS; hv_integrate |

## Open questions

- **Vault token source/freshness** — keychain `vault-admin-token` assumed; it was expired (403)
  during testing. Support minting a short-lived token (use_root) or a configurable token path, or
  just fail clearly when invalid? (Default: read keychain, clear error on failure.)
- **Tunnel reuse** — ephemeral per-operation vs a cached ref-counted tunnel per VM (perf vs
  simplicity). Default: ephemeral; revisit if slow.
- **Field-name reconciliation** — confirm exactly what `cloud_db_config_add` writes vs what the
  toolkit reader expects, and normalize in the reader.
