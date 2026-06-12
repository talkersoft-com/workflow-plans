# Deck: cloud-manager

## Deck name
`cloud-manager`

## Orchestration id (plan codename)
`ansible-p2-decomposition` — Ansible Epic series plan 2 of 9 (executes after P1 is merged).

## Repos with code changes
| Repo | What changes |
|------|-------------|
| `cloud-manager-api` | PlaybookPlay/PlaybookTask/PlaybookHandler/VarBlock/EntityEdge entities (`play` `ptask` `hdlr` `vblk` `edge`), decomposer service, usedBy/graph endpoints, backfill command |
| `cloud-manager-mcp` | New `graph.ts` tools, profile registration, dist rebuild |
| `workflow-plans` | This plan folder |

**Must NOT change:** cloud-manager-web, vorch-lib, vorch-service/porch.

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
         message: "feat: playbook decomposition records + entity graph (P2 — play/ptask/hdlr/vblk/edge)"
         title:   "Ansible Epic P2: decomposed records, edge graph, used-by APIs"
         stage:   "exec"
```
