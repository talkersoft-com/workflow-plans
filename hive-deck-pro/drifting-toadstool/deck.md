# Deck: hive-deck-pro

## Deck name
`hive-deck-pro`

## Repos with code changes
| Repo | What changes |
|------|-------------|
| `hive-deck-pro` | Token engine (`expandPath`), `Setup.DeckConfig` + `layerPaths` per-file resolution, union scans (decks list, workflow registry, fragment chain), HV_PROFILE machine-identity selection, mcps token table, tests, docs |
| `workflow-plans` | This plan folder |

NOT in this plan: workflow-configuration restructuring (base/ migration is a follow-up config
change after the code ships).

## Branch
`drifting-toadstool`

## Initialize (Task 0000)
```
hv_status  deck: "hive-deck-pro"
hv_next    deck: "hive-deck-pro"
```
(or `hv_init` if repos are on the default branch)

## Ship
```
hv_ship  deck: "hive-deck-pro"
         message: "feat: shared config layer + canonical path tokens (R7.3-R7.6)"
         title:   "Shared config layer + WorkspaceRoot tokens (drifting-toadstool)"
         stage:   "exec"
```
