# RabbitMQ Marketplace Blueprint

## What this workflow does
Adds RabbitMQ as a one-click marketplace item in Cloud Manager. A user picks the
RabbitMQ tile and instantiates it; Cloud Manager provisions a new Ubuntu 22.04 VM and
runs an Ansible playbook that installs and configures RabbitMQ (admin user, vhost,
management plugin, password to Vault, ports 5672/15672). Mirrors the existing
`postgres-jammy` blueprint — playbook seed files + one EF migration, no new C# code.

## Read before starting
- `PLAN.md` — objective, design, phases, and open questions
- `deck.md` — which deck and repos are in scope; pre-written hv MCP calls
- `Execution/Exec.md` — full task list and execution instructions (created at approval)
- `../../common/toolkit/hive-deck.md` — hv MCP call patterns
- Reference prior art: `cloud-manager-api/seed/playbooks/pg-14-jammy.yaml` (+ `.meta.json`)
  and migration `20260611040351_AddMarketplaceBlueprints.cs`

## Constraints
- Reuse `.cicd/import-playbooks.py` unchanged — do not modify the importer.
- Reuse the existing `marketplace` feature flag — do not recreate it.
- No new controllers or services — the generic Playbook/Marketplace/Blueprint API
  already handles any playbook/blueprint.
- The EF migration must be reversible (`Down` deletes the seeded rows).
