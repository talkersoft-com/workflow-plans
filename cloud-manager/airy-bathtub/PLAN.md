# Plan: Default-deny host firewall (ufw) for provisioned VMs — v2 (decision-complete)

> **Supersedes the `syrupy-jadeplant` firewall plan**, which left open questions. This version is
> **fully decided** — there is intentionally **no "Open questions" section**. A headless approve/exec
> agent must execute it end-to-end without pausing for operator input; resolve any ambiguity from
> the codebase, never defer.

## Objective

Lock provisioned VMs down at the network layer. Today every VM boots with **no host firewall**, so
any listening service is reachable by anything that can route to it (proven: an off-subnet Mac
reached Postgres `5432` directly — only `pg_hba` stopped auth). After this: a fresh VM is
**SSH-only** by default (port 22 from anywhere; all else denied), and each service playbook opens
**its own** port to the **trusted subnet only**. Repos: `vorch-lib` (cloud-init baseline) +
`cloud-manager-api` (seed playbook YAMLs). No api/web/db-schema changes; no migration.

## Background

- `vorch`'s cloud-init user-data sets up user/netplan/hostname/secret-injection but **no firewall**.
- **Playbook source of truth is git, not the DB.** Canonical playbooks live at
  `cloud-manager-api/seed/playbooks/*.yaml` (+ `.meta.json`); `cloud-manager-mcp/.cicd/import-playbooks.py`
  pushes them into the DB (match by name → POST if missing / PATCH if content differs / no-op if same).
  The controller materializes playbooks from the **DB** at provision, so the flow is: **edit seed YAML
  → run `import-playbooks.py` → DB updated.** Never hand-edit the DB via `cloud_playbook_update`
  (drifts from git, overwritten on next import).
- **Current per-service state (verified):** the Postgres seed playbooks (`pg-14-jammy.yaml`,
  `pg-16-noble.yaml`) have **no ufw task** (they only set `listen_addresses='*'` + `pg_hba_sources`
  `10.0.150.0/24` + `10.0.144.140/32`). The RabbitMQ seed playbooks (`rabbitmq-jammy.yaml`,
  `rabbitmq-noble.yaml`) **already** have a ufw task that opens `5672` + `15672` **to anywhere**
  (no `from_ip`).

## Locked decisions

- **ufw** (ships in the Ubuntu cloud image — no install).
- **SSH (22) allowed from `0.0.0.0/0`** — operator roams networks; parity with future Vultr VMs;
  safe because the cloud image is key-only (no password auth). Final.
- **default deny incoming, allow outgoing.**
- **Service ports → trusted subnet only.** Keep Postgres listening on the subnet (do **not** revert
  to localhost-only) — peer VMs need direct access. Off-subnet/operator DB access is via SSH tunnel
  (the separate manager-provider work).
- **Trusted sources** = `10.0.150.0/24` + `10.0.144.140/32` (reuse the Postgres playbook's
  `pg_hba_sources`, or a `ufw_sources` var with identical values).
- **Only newly provisioned VMs** get the baseline (cloud-init first boot); do **not** retro-fire
  existing VMs.
- **RabbitMQ `15672` (mgmt UI) → trusted subnet** (same treatment as `5672`/Postgres) — consistent,
  simple; operators off-subnet reach it via SSH tunnel.

## Design

### 1. vorch-lib — cloud-init ufw baseline (every VM)
In the user-data builder (`InitializeHost` / `buildUserData`), append to `runcmd` **after netplan
apply**, in this exact lockout-safe order:
```
ufw allow 22/tcp                      # allow SSH BEFORE enabling — never lock out
ufw --force default deny incoming
ufw --force default allow outgoing
ufw --force enable
```
Idempotent; no install. Established SSH (porch post-boot runs) survives (22 allowed + ufw permits
established). Static in the template → **vorch-service unchanged** (no new create-vm field).

### 2. cloud-manager-api seed playbooks — per-service subnet rules
Edit the seed YAML, then sync to the DB:
- **Postgres** (`pg-14-jammy.yaml`, `pg-16-noble.yaml`): **ADD** a ufw task —
  ```yaml
  - name: Allow Postgres from trusted sources
    community.general.ufw:
      rule: allow
      from_ip: "{{ item }}"
      to_port: "5432"
      proto: tcp
    loop: "{{ pg_hba_sources }}"
  ```
- **RabbitMQ** (`rabbitmq-jammy.yaml`, `rabbitmq-noble.yaml`): **TIGHTEN** the existing ufw task by
  adding `from_ip: "{{ item }}"` looped over the trusted sources, so `5672` and `15672` are
  subnet-only instead of open to anywhere.
- Then run `python3 cloud-manager-mcp/.cicd/import-playbooks.py` against the live API → PATCHes the
  DB rows so runtime matches git.

### Access model when done
```
EVERY VM (cloud-init):     22 ✅ from anywhere ; everything else ⛔ deny
+ Postgres VM (pg playbook): 5432 ✅ from 10.0.150.0/24 + 10.0.144.140/32
+ RabbitMQ VM (rabbit pb):   5672 + 15672 ✅ from the same trusted sources
off-subnet → any service port ⛔  (operator reaches services via SSH tunnel)
```

### Conventions
Mirror the existing user-data builder + playbook patterns. No secret/value logging. No
api/web/db-schema changes; no migration.

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000 | Setup | hv_status + hv_init/hv_next; record branch |
| 0001 | vorch-lib — cloud-init ufw baseline | append the lockout-safe ufw block to the user-data `runcmd`; golden/unit test of generated user-data (block present; ordering correct; empty-secrets path otherwise byte-stable) |
| 0002 | seed playbooks — per-service subnet rules | pg-14/pg-16: add ufw 5432→sources; rabbitmq-jammy/noble: add `from_ip` to the existing ufw task (5672+15672→sources); then run `import-playbooks.py` and confirm PATCH/no-op output |
| 0003 | E2E + build + deploy | build vorch; **manual vorch redeploy** on the hypervisor (record cmd); provision a PLAIN VM → SSH ok, other ports blocked from off-subnet; provision a POSTGRES VM → SSH ok, 5432 allowed from an on-subnet peer + blocked from the off-subnet Mac; confirm DB playbook content matches seed YAML; confirm no existing VM changed; write Results + LESSONS; hv_integrate |

## Acceptance criteria (exec verifies — never asks)
- Plain VM: SSH reachable; all other inbound blocked (checked from the off-subnet Mac).
- Postgres VM: SSH ok; `5432` reachable from an on-subnet peer; `5432` blocked from the off-subnet Mac.
- `import-playbooks.py` reports the pg + rabbit playbooks updated; DB matches seed YAML.
- No already-running VM was modified.

## Out of scope
- The database-toolkit `manager` provider / SSH-tunnel tooling (separate plan).
- Reverting Postgres to localhost-only.
- Retroactively firewalling already-running VMs.
