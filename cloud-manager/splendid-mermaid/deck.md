# Deck: cloud-manager

## Deck name
`cloud-manager`

## Repos with code changes
| Repo | What changes |
|------|-------------|
| `cloud-manager-api` | **P0:** reorder `MarketplaceController.Instantiate` to resolve secret bindings BEFORE creating the VM record; 422 (binding name + Vault path, never the value) on missing/unresolvable secret; no orphaned VM; failure-event guard for any post-create failure. **P1:** `InstantiateRequest.secretBindingParams` (per-binding param values); `SecretBindingResolver.ResolveForBlueprintAsync` uses supplied params to substitute `{vm_public_id}` with an operator-chosen SOURCE VM; required-param-missing → 422. |
| `cloud-manager-mcp` | `cloud_marketplace_provision` gains optional `secretBindingParams` passthrough to the instantiate endpoint. |

Repos not listed will be on the feature branch but skipped by hv_integrate.

## Branch
TBD — recorded at runtime in Task 0000.

## Operator action after merge
**Restart cloud-manager-mcp via `/mcp`** (changed tool surface). A full Claude Code restart may be
needed to re-enumerate the cloud-manager MCP tools (the deferred-tool index goes stale on reconnect).

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
              message: "feat: secret-binding provisioning hardening + instantiate param resolution"
              title:   "feat: secret-binding provisioning hardening + instantiate param resolution"
              stage:   "exec"
```
