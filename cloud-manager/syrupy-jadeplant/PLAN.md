# Plan: Default-deny host firewall (ufw) for provisioned VMs

## Objective

Lock down provisioned VMs at the network layer. Today every VM boots with **no host firewall**,
so any listening service is exposed to anything that can route to it (proven: an off-subnet Mac
reached Postgres `5432` directly — only `pg_hba` stopped the auth). This workflow makes a fresh VM
**SSH-only by default**: `ufw` denies all inbound except port 22 (reachable from any network, for
operator mobility + future Vultr parity), and each service playbook opens **its own** port to the
**trusted VM subnet only**. Peer VMs keep direct on-subnet DB access; off-subnet/operator access
goes via SSH tunnel (the separate #6 manager-provider work). This is workflow **#5** of the security
hardening track. **No api/web/db changes; no migration.**

## Background

- **Current state:** vorch's cloud-init user-data sets up the user, netplan, hostname, and (from #2)
  secret injection — but **no firewall**. Postgres VMs run `listen_addresses = '*'` (per the pg-14
  playbook) with `pg_hba` limited to the subnet + KVM host; that `pg_hba` layer is the *only* thing
  protecting an otherwise network-open port.
- **Decision (with operator):** keep Postgres reachable on the subnet (peer VMs need it) — do **not**
  revert to localhost-only. Add a host firewall as the outer layer instead.
- **Operator mobility:** the Mac roams between networks, so SSH (22) must be reachable from anywhere;
  this also matches how future Vultr/cloud VMs will be reached (SSH is the universal path). Safe
  because the cloud image is key-only (no password auth).
- **Scope of effect:** the baseline is applied by **cloud-init at first boot**, so it only affects
  **newly provisioned VMs**. Existing running VMs are untouched by this workflow.

## Design

### 1. vorch-lib — cloud-init ufw baseline (every VM)
In the user-data builder (`InitializeHost` / `buildUserData`), append to the `runcmd` section
(after netplan apply, so networking is up), in this **lockout-safe order**:
```
ufw allow 22/tcp                      # allow SSH BEFORE enabling — never lock out
ufw --force default deny incoming
ufw --force default allow outgoing
ufw --force enable
```
- `ufw` ships in the Ubuntu cloud image — no install. Commands are idempotent.
- Established connections are preserved by ufw, and 22 is allowed, so the post-boot porch/ansible
  SSH runs are unaffected.
- No new create-vm field — the baseline is static in the template, so **vorch-service is unchanged**.

### 2. Service playbooks — open their own port to the trusted subnet
Each service playbook that exposes a port adds a ufw task looped over the trusted sources (reuse the
Postgres playbook's existing `pg_hba_sources`, or a parallel `ufw_sources` var with the same values
`10.0.150.0/24` + `10.0.144.140/32`):
```yaml
- name: Allow <svc> from trusted sources
  community.general.ufw:
    rule: allow
    from_ip: "{{ item }}"
    to_port: "<svc-port>"
    proto: tcp
  loop: "{{ ufw_sources }}"
```
- **Postgres** (`pb_PG14JAMMY01`, `pb_EMEJCH08GG`): port `5432`.
- **RabbitMQ** (jammy/noble): its broker/mgmt ports (confirm exact set during exec — typically
  `5672`, and `15672` if the mgmt UI is wanted on-subnet).
- The installing playbook owns its firewall rule, mirroring how it already owns its `pg_hba` rules.

### Access model when done
```
22/tcp  : ALLOW from anywhere        → operator/MCP reach every VM via SSH (any network)
5432    : ALLOW from 10.0.150.0/24   → peer VMs reach Postgres directly on-subnet
*       : DENY incoming              → nothing else exposed; off-subnet 5432 is closed
```

### Conventions
Mirror the existing user-data builder + playbook patterns. No secret/value logging (injection logic
unchanged). No api/web/db/migration changes.

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000 | Setup | hv_status + hv_init/hv_next; record branch |
| 0001 | vorch-lib — cloud-init ufw baseline | append lockout-safe ufw block (allow 22 → default deny in / allow out → enable) to the user-data runcmd; golden/unit test of generated user-data; empty-secrets path still byte-stable apart from the ufw block |
| 0002 | Service playbooks — per-port subnet allow | Postgres (pg14/pg16) + RabbitMQ playbooks add `ufw allow from {{ ufw_sources }} to <svc-port>`; mirror `pg_hba_sources` |
| 0003 | E2E + build + deploy | provision a **plain** VM → SSH reachable, all other ports blocked; provision a **Postgres** VM → SSH ok, 5432 allowed from subnet / blocked off-subnet (verify from the off-subnet Mac and an on-subnet peer); build vorch; **manual vorch redeploy** on the hypervisor; apply playbook ufw rules; write Results + LESSONS; hv_integrate |

## Open questions

- **SSH from `0.0.0.0/0`** — accepted (key-only auth + operator roaming + Vultr parity). Alternative
  (restrict to known admin ranges) rejected because the Mac switches networks.
- **Canonical playbook source** — are the Postgres/RabbitMQ playbooks DB-only (edit via
  `cloud_playbook_update`) or seeded from a repo? Determines how the per-service ufw rule is
  persisted; exec records which.
- **RabbitMQ ports** — confirm the exact set to open to the subnet (5672 always; 15672 mgmt only if
  desired) from the rabbitmq playbook during exec.
