# Deck: cloud-manager

## Deck name
`cloud-manager`

## Repos with code changes
| Repo | What changes |
|------|-------------|
| `cloud-manager-api` | `SecretBinding` + `BlueprintSecretBinding` EF entities; one reversible migration creating `marketplace.secret_bindings` + `marketplace.blueprint_secret_bindings`; CRUD + attach/detach/reorder/list endpoints (camelCase, public_ids, soft-delete; mirror `blueprint_playbooks`). No instantiate changes. |
| `cloud-manager-mcp` | Tools: `cloud_secret_binding_list/get/create/update/delete`, `cloud_blueprint_secret_binding_attach/detach/list`. |
| `cloud-manager-web` | "Secret Bindings" management page + params editor (path-template placeholder scan + source picker); blueprint "Attached Secrets" attach/detach/reorder UI. No instantiate-form changes. |

Repos not listed will be on the feature branch but skipped by hv_integrate.

## Branch
`sour-lemonade`

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
              message: "feat: Secret Bindings — authoring foundation"
              title:   "feat: Secret Bindings — authoring foundation"
              stage:   "exec"
```
