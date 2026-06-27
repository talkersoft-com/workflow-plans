# Deck: cloud-manager

## Deck name
`cloud-manager`

## Repos with code changes
| Repo | What changes |
|------|-------------|
| `cloud-manager-api` | One reversible migration adding nullable `resolved_vault_path` to `vm.vm_secret_bindings`; populate it from the `SecretBindingResolver`-computed path when the usage row is inserted at provision; `GET /api/v1/vm/{publicId}/secret-binding` returns the stored `resolvedVaultPath` (recompute only when null, for pre-change rows). Path only, never the value. |
| `cloud-manager-mcp` | Fix `apiRequest` camelCase guard to exempt the inner `secretBindingParams[].params` map keys (free-form placeholder names like `vm_public_id`); guard unchanged for all other/wire fields. |

Repos not listed will be on the feature branch but skipped by hv_integrate.

## Branch
TBD — recorded at runtime in Task 0000.

## Operator action after merge
**Rebuild the LOCAL cloud-manager-mcp dist (`npm run build`) + `/mcp`** so Claude Code picks up the
fixed tool (the server-side deploy does not rebuild the dev MCP; `/mcp` alone relaunches stale dist).

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
              message: "fix: secret-binding verification — MCP param guard + persisted resolved path"
              title:   "fix: secret-binding verification — MCP param guard + persisted resolved path"
              stage:   "exec"
```
