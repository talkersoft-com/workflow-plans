# Deck: cloud-manager

## Deck name
`cloud-manager`

## Repos with code changes
| Repo | What changes |
|------|-------------|
| `database-toolkit` | Wire `resolveVaultEntries` into the config-loader path (`tools/shared/database-config-loader.ts`), rebuild dist |
| `workflow-plans` | Findings documents from Phases 0002 and 0003 land in this plan folder |

Repos not listed will be on the feature branch but skipped by hv_ship. Phase 0002 may add a change to the repo owning the postgres playbook/role **only if** the investigation confirms one owns postgres setup.

## Branch
`beaming-messenger`

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
hv_ship  deck: "cloud-manager"
         message: "<commit message>"
         title:   "<PR title>"
```
