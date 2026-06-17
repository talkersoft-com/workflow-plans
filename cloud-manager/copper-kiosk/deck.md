# Deck: cloud-manager

## Deck name
`cloud-manager`

## Repos with code changes
| Repo | What changes |
|------|-------------|
| `cloud-manager-web` | Marketplace storefront page, Blueprints builder pages, VM detail provenance + run-chain panel, api client + Redux slices, nav + feature flag |
| `workflow-plans` | This plan folder + the Phase 0005 findings doc |

Repos not listed will be on the feature branch but skipped by hv_ship. **vorch-lib, vorch-service,
porch, cloud-manager-api, and cloud-manager-mcp must not change in this plan** — Phase 0005 may
only produce findings and a follow-up plan draft for the `vorch@cloud-manager` workflow.

## Branch
`copper-kiosk` (plan codename; written on planning branch `lucky-engineblock` — series plan 2,
executes after plan 1 is approved and executed)

## Initialize (Task 0000)
```
hv_status  deck: "cloud-manager"
hv_next    deck: "cloud-manager"
```

## Ship
```
hv_ship  deck: "cloud-manager"
         message: "<commit message>"
         title:   "<PR title>"
         stage:   "exec"
```
