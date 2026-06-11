# Deck: cloud-manager

## Deck name
`cloud-manager`

## Repos with code changes
| Repo | What changes |
|------|-------------|
| `cloud-manager-web` | BlueprintsPage row Edit affordance; create-modal success navigation |
| `workflow-exec` | Orchestration folder (runtime) |

## Branch
`wired-rooftop` (plan codename)

## Initialize (Task 0000)
```
hv_status  deck: "cloud-manager"
hv_next    deck: "cloud-manager"
```

## Ship
```
hv_ship  deck: "cloud-manager"
         message: "fix: blueprint composer navigation — list edit affordance + create lands on composer"
         title:   "Blueprint composer navigation (wired-rooftop)"
         stage:   "exec"
```
