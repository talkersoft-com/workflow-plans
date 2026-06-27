# Cloud Manager — Default-deny host firewall (ufw) for provisioned VMs

## What this workflow does
Workflow **#5** (security hardening): provisioned VMs currently boot with **no host firewall** — any
listening service is network-exposed (an off-subnet Mac reached Postgres 5432 directly). This adds a
**ufw default-deny baseline** at cloud-init first boot so a fresh VM is **SSH-only** (port 22 from
anywhere; everything else denied), and each **service playbook opens its own port to the trusted VM
subnet only**. Peer VMs keep direct on-subnet DB access; off-subnet/operator access uses SSH tunnels
(the separate #6 work). vorch-lib (cloud-init) + service playbooks; no api/web/db changes.

## Read before starting
- `deck.md` — repos in scope; the manual vorch redeploy + playbook-update steps
- `Execution/Exec.md` — full task list and execution instructions
- `PLAN.md` — decisions, the access model, ufw ordering, phases, open questions
- `../../common/toolkit/hive-deck.md` — hv MCP call patterns

## Constraints
- **Lockout-safe ordering is MANDATORY:** `ufw allow 22/tcp` **before** `ufw --force enable`; use
  `--force` for non-interactive. Getting this wrong locks SSH out of every new VM.
- **Only newly provisioned VMs** get the baseline (cloud-init first boot). Do **not** retro-fire ufw
  on existing running VMs in this workflow.
- **Keep Postgres on the subnet** — do NOT revert listen/pg_hba to localhost-only (peer VMs need it).
  The firewall is the outer layer on top of the existing pg_hba.
- SSH (22) is intentionally open from `0.0.0.0/0` (key-only auth; operator roams; Vultr parity).
- ufw ships in the Ubuntu image — no install. All commands idempotent.
- Mirror the existing user-data builder + playbook patterns. **No api/web/db changes; no migration.**
- **vorch-service unchanged** (the baseline is static in the user-data template).
- Established SSH must survive `ufw enable` (22 allowed + ufw permits established) so porch/ansible
  post-boot runs are unaffected.

## Out of scope
- The #6 `manager` DB provider / SSH-tunnel tooling.
- Reverting Postgres to localhost-only.
- Retroactively firewalling already-running VMs.
