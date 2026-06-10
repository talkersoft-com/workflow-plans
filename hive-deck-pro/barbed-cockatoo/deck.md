# Deck: hive-deck-pro

## Deck name
`hive-deck-pro`

## Repos with code changes
| Repo | What changes |
|------|-------------|
| `hive-deck-pro` | Builtin plan fallback in `ops/workflow.go` (plan type only); MCP TS fixes in `mcp/src/tools/workflow_plan.ts` (field name + default) plus contract audit of other tools; rebuilt `mcp/dist`. Note: branch already carries an unpushed test-fix commit (fe1e6bc) that ships with this PR |
| `workflow-configuration` | New `decks/workflows/plan/cloud-manager/plan.yaml` (empty extension workaround, becomes redundant after Fix 2 but harmless) |
| `workflow-plans` | This plan folder |

Repos not listed will be on the feature branch but skipped by hv_ship.

## Branch
`barbed-cockatoo`

## Initialize (Task 0000)
```
hv_status  deck: "hive-deck-pro"
hv_init    deck: "hive-deck-pro"
```
If already provisioned on a prior feature branch with all PRs merged:
```
hv_status  deck: "hive-deck-pro"
hv_next    deck: "hive-deck-pro"
```

## Ship
```
hv_ship  deck: "hive-deck-pro"
         message: "<commit message>"
         title:   "<PR title>"
```
