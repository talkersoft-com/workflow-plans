# Deck: hive-deck-pro

## Deck name
`hive-deck-pro`

## Repos with code changes
| Repo | What changes |
|------|-------------|
| `hive-deck-pro` | Go source: config, resolve, provision, branch, ship, ops/init |
| `workflow-configuration` | config.yaml cleanup; `branch: hive` added to all deck YAMLs |

## Branch
`oaty-gibbon`

## Initialize (Task 0000)
```
hv_status  deck: "hive-deck-pro"
```
Already on oaty-gibbon — no init or next needed.

## Ship
```
hv_ship  deck: "hive-deck-pro"
         message: "add per-deck branch field, remove global target branch config"
         title:   "Add per-deck branch field, remove global target branch config"
```
