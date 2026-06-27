# Deck: cloud-manager

## Deck name
`cloud-manager`

## Repos with code changes
| Repo | What changes |
|------|-------------|
| `cloud-manager-api` | New `GET /api/v1/vm/{publicId}/secret-binding` on `VirtualMachineController` + backing service method (query `vm_secret_bindings` joined to `secret_bindings`; returns `{publicId, credName, secretBindingId, secretBindingName, resolvedVaultPath, createdAt}`; **never the value**; 404 if VM missing; camelCase; public_ids). No DB migration. |
| `cloud-manager-mcp` | `cloud_vm_secret_binding_list` (wraps the new endpoint — verification tool); `cloud_blueprint_secret_binding_update` (wraps existing `PATCH /blueprint/{id}/secret-binding/{bsbId}` — credName/position). |

Repos not listed will be on the feature branch but skipped by hv_integrate.

## Branch
TBD — recorded at runtime in Task 0000.

## Operator action after merge
**Restart cloud-manager-mcp via `/mcp`** to pick up the two new tools (the running server keeps the
old tool list until reloaded). Surface this in `Results/RESULT.md`.

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
              message: "feat: VM secret-binding read endpoint + MCP gap tools"
              title:   "feat: VM secret-binding read endpoint + MCP gap tools"
              stage:   "exec"
```
