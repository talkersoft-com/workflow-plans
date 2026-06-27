# Cloud Manager — `manager` database provider for database-toolkit MCP

## What this workflow does
Workflow **#6** (security track): add a `manager` provider to the database-toolkit MCP so `db_*`
tools connect to cloud-manager VMs with **no creds in local config** — DB creds resolved from Vault
and reached over an **SSH tunnel** to the VM's localhost. Closes the live integration gap:
`cloud_db_config_add` already registers a Vault/SSH entry, but the toolkit couldn't consume it
(`db_test_connection` wanted literal creds from a different config path). Keep the existing
`standard` (literal-creds) provider. `database-toolkit` repo only; no api/web changes.

## Read before starting
- `deck.md` — repo in scope; the local-rebuild + Vault-token operator steps
- `Execution/Exec.md` — full task list and execution instructions
- `PLAN.md` — the gap, provider design, Vault paths, SSH-tunnel resolution, phases, open questions
- `../../common/toolkit/hive-deck.md` — hv MCP call patterns

## Constraints
- **Backward compatible:** `standard` configs (literal `host/port/username/rootPassword`) behave
  exactly as today; only `provider:manager` entries trigger Vault/SSH resolution.
- **No secret values in output or logs** — the Vault password and SSH private key are used
  in-process only, never printed (preserve the existing "no row data / no secrets" behavior).
- **Read the shared config** `~/.hv/toolkit/db/config/database.json` (where `cloud_db_config_add`
  writes) so registered DBs are discoverable + usable by every `db_*` tool.
- **manager resolution:** Vault creds (`cloudmanager/data/vm/instances/{vmId}/postgres`) + SSH key
  (`.../ssh`) → SSH tunnel `cloudmanager@<host>` `-L <local>:127.0.0.1:<port>` → connect. **Tear down
  tunnels** (no leaked ports/procs).
- Works **with or without** the #5 firewall (tunnel to VM localhost is subnet-independent).
- **Do NOT change `cloud_db_config_add`** (it already writes the entry); reconcile any field mismatch
  in the toolkit's reader. No api/web changes; Postgres only for this cut.
- After merge: **rebuild local database-toolkit + `/mcp`**; ensure a valid keychain Vault token.

## Out of scope
- The #5 firewall; changes to `cloud_db_config_add`; non-Postgres engines; a long-lived tunnel daemon.
