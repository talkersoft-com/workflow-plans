# Cloud Manager — Default-deny host firewall (ufw) for provisioned VMs (v2)

## What this workflow does
Security hardening: provisioned VMs boot with **no host firewall** today (an off-subnet Mac reached
Postgres 5432 directly). This adds a **ufw default-deny baseline** at cloud-init first boot so a
fresh VM is **SSH-only** (22 from anywhere; everything else denied), and edits the **seed playbooks**
so each service opens its own port to the **trusted subnet only** (Postgres adds 5432; RabbitMQ
tightens its existing 5672+15672 rule). `vorch-lib` (cloud-init) + `cloud-manager-api` (seed YAML),
synced to the DB via `import-playbooks.py`. No api/web/db-schema changes.

> **Supersedes the `syrupy-jadeplant` firewall plan.** This version is **decision-complete** — there
> is no "Open questions" section. Execute end-to-end without pausing for operator input; resolve any
> ambiguity from the codebase (seed YAML, the existing rabbitmq ufw task, `pg_hba_sources`).

## Read before starting
- `deck.md` — repos in scope; the mandatory `import-playbooks.py` sync + manual vorch redeploy
- `Execution/Exec.md` — full task list and execution instructions
- `PLAN.md` — locked decisions, access model, ufw ordering, phases, acceptance criteria
- `../../common/toolkit/hive-deck.md` — hv MCP call patterns

## Constraints (all decided — do not defer to the operator)
- **Lockout-safe ordering is MANDATORY:** `ufw allow 22/tcp` **before** `ufw --force enable`. Getting
  this wrong bricks SSH on every new VM.
- **SSH (22) open from `0.0.0.0/0`** (key-only auth; operator roams; Vultr parity) — final.
- **Service ports → trusted subnet only** (`10.0.150.0/24` + `10.0.144.140/32`). Keep Postgres on
  the subnet (do NOT revert to localhost-only). RabbitMQ 15672 is subnet-only too.
- **Playbook edits go in the seed YAML (git source of truth), then `import-playbooks.py` syncs the
  DB.** Never hand-edit the DB via `cloud_playbook_update`.
- **Only newly provisioned VMs** get the baseline; do not retro-fire existing VMs.
- Mirror existing user-data builder + playbook patterns. No secret/value logging. **No
  api/web/db-schema changes; no migration. vorch-service unchanged.**
- Established SSH must survive `ufw enable` so porch post-boot runs are unaffected.

## Out of scope
- The database-toolkit `manager` provider / SSH-tunnel tooling.
- Reverting Postgres to localhost-only.
- Retroactively firewalling already-running VMs.
