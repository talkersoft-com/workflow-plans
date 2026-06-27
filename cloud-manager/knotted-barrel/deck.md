# Deck: cloud-manager

## Deck name
`cloud-manager`

## Repos with code changes
| Repo | What changes |
|------|-------------|
| `database-toolkit` | Add a `provider` field to config entries (`standard` default = literal creds, unchanged; `manager` = `{vmId, postgresVersion}`, no creds). Read the shared `~/.hv/toolkit/db/config/database.json`. `manager` resolution at connect time: fetch DB creds + SSH key from Vault, open an SSH tunnel to the VM's localhost:5432, connect — transparent to all `db_*` tools. No secret values logged. |

Repos not listed will be on the feature branch but skipped by hv_integrate.

## Branch
TBD — recorded at runtime in Task 0000.

## Operator steps after merge
- **Rebuild the LOCAL database-toolkit** (dist/binary) and `/mcp` reload — Claude Code runs the
  toolkit from the local build; server-side deploy / `/mcp` alone won't pick up new code (same
  stale-build gotcha as cloud-manager-mcp).
- Ensure a **valid Vault token** is in the keychain (`vault-admin-token`) — it was observed expired
  during testing; the `manager` provider needs a working token to read Vault.

## Initialize (Task 0000)
```
hv_status  deck: "cloud-manager"
hv_init    deck: "cloud-manager"
```
If already provisioned on a prior feature branch with all PRs merged:
```
hv_status  deck: "cloud-manager"
hv_next    deck: "cloud-manager"
```

## Ship
```
hv_integrate  deck: "cloud-manager"
              message: "feat: database-toolkit manager provider (Vault creds + SSH tunnel, no creds in config)"
              title:   "feat: database-toolkit manager provider (Vault creds + SSH tunnel, no creds in config)"
              stage:   "exec"
```
