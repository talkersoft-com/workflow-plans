# Deck: cloud-manager

## Deck name
`cloud-manager`

## Repos with code changes
| Repo | What changes |
|------|-------------|
| `cloud-manager-api` | `Secret` + `VmSecretBinding` EF entities; nullable `secret_id` FK on `SecretBinding`; **one reversible migration** creating `marketplace.secrets`, `vm.vm_secret_bindings`, the `secret_bindings.secret_id` column, both `ON DELETE RESTRICT` FKs, and the `vault_path` CHECK; Secret CRUD endpoints (create = Vault write + Pending→Active state machine; delete = guard→Deleting→Vault hard-delete→Deleted; list; get). camelCase, public_ids, soft-delete; mirror `secret_bindings`. No provisioning/instantiate changes. |
| `cloud-manager-mcp` | Tools `cloud_secret_list / get / create / delete`. |
| `cloud-manager-web` | **Secret Manager** page (list, fixed-prefix `cloudmanager/data/manual/` create form, guarded delete); Secret Bindings page gains "reference managed Secret" (static binding) while keeping templated authoring. |

Repos not listed will be on the feature branch but skipped by hv_integrate.

## Branch
`wheezy-cottonmouth`

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
              message: "feat: Secret Manager — CM-owned secrets + referential-integrity model"
              title:   "feat: Secret Manager — CM-owned secrets + referential-integrity model"
              stage:   "exec"
```
