# Deck: cloud-manager

## Deck name
`cloud-manager`

## Repos with code changes
| Repo | What changes |
|------|-------------|
| `vorch-lib` | cloud-init user-data builder (`InitializeHost`/`buildUserData`): append a **lockout-safe ufw baseline** to `runcmd` after netplan apply — `ufw allow 22/tcp` → `ufw --force default deny incoming` → `ufw --force default allow outgoing` → `ufw --force enable`. Makes new VMs SSH-only by default. No new create-vm field (vorch-service unchanged). |

Repos not listed will be on the feature branch but skipped by hv_integrate.

## Branch
TBD — recorded at runtime in Task 0000.

## Manual / data steps
- **Service-playbook ufw rules (NOT a repo change if playbooks are DB-stored):** Postgres
  (`pb_PG14JAMMY01`, `pb_EMEJCH08GG`) + RabbitMQ playbooks add `ufw allow from {{ ufw_sources }} to
  <svc-port>` (5432 for pg; rabbit ports TBC). Apply via `cloud_playbook_update` (or the
  playbook-source repo if one exists) — exec records which and the exact content.
- **vorch redeploy is manual** (hypervisor `ubuntu-server`) — the deploy script targets api/mcp/web
  only; rebuild + restart `vorch_service` on the host and record the command in `Results/RESULT.md`.

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
              message: "feat: default-deny ufw firewall on provisioned VMs (SSH-only baseline + per-service subnet allow)"
              title:   "feat: default-deny ufw firewall on provisioned VMs (SSH-only baseline + per-service subnet allow)"
              stage:   "exec"
```
