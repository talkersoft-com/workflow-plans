# Deck: cloud-manager

## Deck name
`cloud-manager`

## Orchestration id (plan codename)
`ansible-p3-roundtrip` — Ansible Epic series plan 3 of 9 (executes after P2 is merged).

## Repos with code changes
| Repo | What changes |
|------|-------------|
| `cloud-manager-api` | `roundtrip-service/` Python sidecar (FastAPI + ruamel), YamlDocument entity (`ydoc`) + migration, proxy routes `api/v1/roundtrip/*`, structured-edit PATCH endpoints, write-time fidelity check |
| `workflow-plans` | This plan folder |

**Must NOT change:** cloud-manager-mcp, cloud-manager-web, vorch-lib, vorch-service/porch — the
sidecar decision (PLAN.md) exists precisely so porch stays untouched.

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
         message: "feat: ruamel round-trip sidecar + ydoc fidelity layer (P3)"
         title:   "Ansible Epic P3: round-trip YAML service (decompose/recompose, byte-identical)"
         stage:   "exec"
```
