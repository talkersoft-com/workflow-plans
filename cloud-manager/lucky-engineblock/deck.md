# Deck: cloud-manager

## Deck name
`cloud-manager`

## Repos with code changes
| Repo | What changes |
|------|-------------|
| `cloud-manager-api` | Blueprint/BlueprintPlaybook/BlueprintEvent entities, VM.BlueprintId, migration, seeds, Blueprint+Marketplace controllers/services, run sequencer |
| `cloud-manager-mcp` | New `marketplace.ts` tools, profile registration, dist rebuild |
| `workflow-plans` | This plan folder |

(cloud-manager-web is deliberately NOT in this plan — the UI ships in the follow-up plan.)

Repos not listed will be on the feature branch but skipped by hv_ship. **vorch-lib, vorch-service,
and porch must not change** (deck convention; the design requires no changes there).

## Branch
`lucky-engineblock`

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
         stage:   "exec"
```
