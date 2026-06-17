# Deck: cloud-manager

## Deck name
`cloud-manager`

## Repos with code changes
| Repo | What changes |
|------|-------------|
| `cloud-manager-api` | New seed playbook `seed/playbooks/rabbitmq-jammy.yaml` + `.meta.json`; new EF migration under `src/Models/CloudManager.Entities/Migrations/` seeding the `rabbitmq-jammy` blueprint + blueprint_playbooks link. |

Repos not listed will be on the feature branch but skipped by hv_ship.
`cloud-manager-mcp/.cicd/import-playbooks.py` is **reused unchanged** (run at verify time).

## Branch
`tigerish-deli`

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
         message: "feat: RabbitMQ marketplace blueprint"
         title:   "feat: RabbitMQ marketplace blueprint"
```
