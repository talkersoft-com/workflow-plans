# Findings 0002 — Postgres VM provisioning gap (pg-test / vm_SMAZ3C2YVZ)

Investigation deliverable for Phase 2 of `PLAN.md`. Executed 2026-06-10 in workflow `workflow-exec/cloud-manager/gloppy-snapper`.

## Question

PostgreSQL 14 on the pg-test VM came up bound to 127.0.0.1 with a localhost-only pg_hba (fixed by hand on 2026-06-10). Does a playbook own postgres setup on that VM — and if so, should it configure network access?

## Answer: a playbook owns it, and the gap was structural

**Playbook `pg-14-jammy` (`pb_PG14JAMMY01`) installed postgres on vm_SMAZ3C2YVZ.** Evidence:

- `cloud_playbook_run_list` filtered to `vm_SMAZ3C2YVZ` shows exactly 3 runs, all of `pb_PG14JAMMY01` on 2026-06-07: `run_Z92WMH3Z0N` (Failed, ~1s), `run_09VJVBA33R` (Failed, ~82s), `run_BER4PVXPN9` (**Succeeded**, revision `29887398-6f46-43ae-bea3-6538bbe9bdd2`).
- The playbook content (live, API database) installs `postgresql-14` from the default Ubuntu apt repo, creates the app database/user, and writes credentials to Vault KV v2 at `cloudmanager/vm/instances/{vm_public_id}/postgres` — the exact path the `pg-test` db config entry references.
- `cloud_vm_playbook_list` for the VM returns `[]` — no playbook is currently *attached*; the runs were one-shot applies. Attachment state is irrelevant to ownership of what got installed.

**Root cause of the unreachability:** the playbook had **no `listen_addresses` or pg_hba configuration at all**, so Ubuntu's postgres defaults applied (listen on 127.0.0.1, localhost-only pg_hba). Meanwhile the same playbook writes `host: {{ ansible_default_ipv4.address }}` (the VM's real IP) into Vault — i.e. the credentials it publishes advertise an endpoint the server was never configured to listen on. Every postgres VM provisioned by this playbook (and its sibling `pg-16-noble`, which has the identical structure) would reproduce the gap. This was not a one-off fluke.

## pg_hba scope decision (PLAN.md open question)

**Both the VM subnet and the KVM host source IP**, scram-sha-256 only:

| Source | Why |
|--------|-----|
| `10.0.150.0/24` | VM-to-VM access within the guest network |
| `10.0.144.140/32` | SSH-tunnel traffic to guests arrives from the KVM host bridge address (observed live during the 2026-06-10 connectivity session) — subnet-only would silently break the tunnel path that the database-toolkit's `host: localhost` override depends on |

Subnet-only was rejected because the primary consumer today (the database-toolkit MCP through `dev_ssh_tunnel_create`) reaches postgres via the host, not from inside `10.0.150.0/24`.

## Change implemented (conditional fired: playbook owns postgres)

Added to **both** `pg-14-jammy` and `pg-16-noble`, between package install and the service-start task:

1. `listen_addresses = '*'` via `lineinfile` on `/etc/postgresql/{{ pg_version }}/main/postgresql.conf` (matches the manual fix; sources are constrained by pg_hba, not by the bind address) — exposed as var `pg_listen_addresses`.
2. pg_hba `host all all <source> scram-sha-256` rules via `community.postgresql.postgresql_pg_hba` looped over var `pg_hba_sources` (defaults above).
3. A conditional `service: postgresql state: restarted` that fires only when either of the above reported `changed` — idempotent; reruns are no-ops.

Properties: idempotent (lineinfile regexp + pg_hba module + conditional restart); affects newly provisioned VMs only (playbook updates create new revisions — `cloud_playbook_update` changeSummary records the rationale; existing VMs are untouched unless an operator re-runs the playbook, which is safe because the tasks match the manual fix already applied to pg-test).

Where the change lives:

- **Live playbooks** (API database, the operative copies): updated via `cloud_playbook_update` on `pb_PG14JAMMY01` and `pb_EMEJCH08GG`, 2026-06-10.
- **Repo seeds**: `cloud-manager-api/seed/playbooks/pg-14-jammy.yaml` and `pg-16-noble.yaml` rewritten to match.

## Side-finding: seed files had drifted from the live playbooks

Both seed files in `cloud-manager-api/seed/playbooks/` were stale revisions — they still wrote to the old `playbook-secrets/{run_public_id}/...` Vault path and regenerated the password on every run, while the live DB playbooks had moved to the VM-anchored `vm/instances/{vm_public_id}/postgres` path with reuse-existing-password idempotency. Nothing in the API repo reads `seed/playbooks/` programmatically (no seeder code references it), so drift is silent. The seeds are now synced to the live content as part of this change. Recommendation: either add a seed-sync check, or treat the DB as the sole source of truth and demote the seed folder to documentation — candidate for a future plan.
