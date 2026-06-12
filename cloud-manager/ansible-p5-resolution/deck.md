# Deck: cloud-manager

## Deck name
`cloud-manager`

## Orchestration id (plan codename)
`ansible-p5-resolution` — Ansible Epic series plan 5 of 9 (executes after P1, P2, P4 are merged).

## Repos with code changes
| Repo | What changes |
|------|-------------|
| `cloud-manager-api` | Precedence engine service, C-RES preview endpoint, ResolutionSnapshot entity (`rsnap`) + migration, enforceRequired 422 path, run resolution read endpoint |
| `cloud-manager-mcp` | New `resolution.ts` tool, profile registration, dist rebuild |
| `workflow-plans` | This plan folder |

**Must NOT change:** cloud-manager-web, vorch-lib, vorch-service/porch — the engine computes
API-side; porch keeps receiving exactly the merged inputs it receives today.

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
         message: "feat: precedence resolution engine + provenance + gaps (P5 — rsnap)"
         title:   "Ansible Epic P5: effective values, provenance, gap detection"
         stage:   "exec"
```
