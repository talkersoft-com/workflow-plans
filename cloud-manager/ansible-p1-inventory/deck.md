# Deck: cloud-manager

## Deck name
`cloud-manager`

## Orchestration id (plan codename)
`ansible-p1-inventory` — Ansible Epic series plan 1 of 9 (authored on planning branch by
windward-steamboat; executes after the series PR is approved).

## Repos with code changes
| Repo | What changes |
|------|-------------|
| `cloud-manager-api` | Inventory/InventoryGroup/InventoryHost/InventoryGroupMembership/GroupVar/HostVar/InventoryEvent entities (`inv` `invg` `invh` `igm` `gvar` `hvar` `invev`), migration, `ansible-studio` flag seed, services/DTOs/controllers, event service |
| `cloud-manager-mcp` | New `inventory.ts` tools, profile registration, dist rebuild |
| `workflow-plans` | This plan folder |

**Must NOT change:** cloud-manager-web, vorch-lib, vorch-service/porch — the design requires no
changes there.

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
         message: "feat: ansible inventories as records (P1 — inv/invg/invh/igm/gvar/hvar/invev)"
         title:   "Ansible Epic P1: inventory entities, APIs, events, MCP tools"
         stage:   "exec"
```
