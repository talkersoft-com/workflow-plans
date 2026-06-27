# Deck: cloud-manager

## Deck name
`cloud-manager`

## Repos with code changes
| Repo | What changes |
|------|-------------|
| `vorch-lib` | cloud-init user-data builder (`InitializeHost`/`buildUserData`): append a **lockout-safe ufw baseline** to `runcmd` after netplan apply — `ufw allow 22/tcp` → `ufw --force default deny incoming` → `ufw --force default allow outgoing` → `ufw --force enable`. New VMs are SSH-only by default. No new create-vm field (vorch-service unchanged). |
| `cloud-manager-api` | Seed playbooks under `seed/playbooks/`: **Postgres** (`pg-14-jammy.yaml`, `pg-16-noble.yaml`) ADD a ufw task `allow 5432 from {{ pg_hba_sources }}`; **RabbitMQ** (`rabbitmq-jammy.yaml`, `rabbitmq-noble.yaml`) TIGHTEN the existing ufw task with `from_ip: {{ item }}` so 5672+15672 are subnet-only. (Seed YAML is the git source of truth.) |

Repos not listed will be on the feature branch but skipped by hv_integrate.

## Branch
TBD — recorded at runtime in Task 0000.

## Required deploy/sync steps (defined — not optional, not open)
- **Sync seed playbooks → DB:** after editing the seed YAML, run
  `python3 cloud-manager-mcp/.cicd/import-playbooks.py` against the live API (matches by name → PATCH
  changed playbooks). The controller materializes playbooks from the DB, so this step is mandatory.
  Do NOT hand-edit the DB via `cloud_playbook_update` (drifts from git).
- **vorch redeploy is manual** (hypervisor `ubuntu-server`) — the deploy script targets api/mcp/web
  only; rebuild + restart `vorch_service` on the host and record the exact command in `Results/RESULT.md`.

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
hv_integrate  deck: "cloud-manager"
              message: "feat: default-deny ufw on provisioned VMs (SSH-only baseline + seed-playbook subnet rules)"
              title:   "feat: default-deny ufw on provisioned VMs (SSH-only baseline + seed-playbook subnet rules)"
              stage:   "exec"
```
