# Deck: cloud-manager

## Deck name
`cloud-manager`

## Orchestration id (plan codename)
`windward-steamboat`

## Repos with code changes
| Repo | What changes |
|------|-------------|
| `workflow-plans` | ANSIBLE-EPIC-CONTRACTS.md, ANSIBLE-EPIC-LEDGER.md (master-plans/), nine plan folders (cloud-manager/ansible-p1…p9) |
| `workflow-exec` | This orchestration folder (runtime) |

**Must NOT change:** any code repo — this run produces documents only.

## Branch
TBD — created by Task 0000 at execution time; record the generated name here.

## Initialize (Task 0000)
```
hv_status  deck: "cloud-manager"
hv_next    deck: "cloud-manager"
```

## Ship
```
hv_ship  deck: "cloud-manager"
         message: "planning: ansible epic series (P1-P9) + contracts + ledger"
         title:   "Ansible Epic plan series (windward-steamboat)"
         stage:   "plan"
```
