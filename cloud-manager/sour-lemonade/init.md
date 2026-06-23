# Cloud Manager — Secret Bindings (authoring foundation)

## What this workflow does
Adds **Secret Bindings** — a reusable, parameterized definition of a Vault secret that attaches
to marketplace blueprints, mirroring how Playbooks attach. This workflow is the authoring half:
an operator can create Secret Bindings (a no-param "manual" one and a `{vm_public_id}` "postgres"
one), attach them to blueprints, and round-trip them via API + MCP + web. Provisioning-time
resolution and cloud-init/systemd-creds injection are a SEPARATE follow-up workflow; only their
schema extension points (`params[].source`, `injection_provider`, `cred_name`) land here.

## Read before starting
- `deck.md` — deck + repos in scope; pre-written hv MCP calls
- `Execution/Exec.md` — full task list and execution instructions
- `../../common/toolkit/hive-deck.md` — hv MCP call patterns
- `PLAN.md` — objective, data model, the params-as-indicator mechanic, phases

## Constraints
- **Term is "Secret Binding"** everywhere (entity, table, API, MCP tools, UI labels).
- camelCase JSON on the wire; **public_ids only** (never leak Guids); **soft-delete authoritative**.
- The migration is **reversible** (clean up/down), applied BEFORE the api restart on deploy.
- Mirror the existing `blueprint_playbooks` code paths — do not invent a parallel paradigm.
- **OUT OF SCOPE:** instantiate-time resolution, the parameter-source resolver, the injection
  provider, reading Vault, cloud-init injection, create-vm changes. Add the schema fields they
  need (`params[].source`, `injection_provider`, `cred_name`) but DO NOT exercise them.
- Both canonical bindings must round-trip: manual (`/manual/secret/i/stored`, no params) and
  postgres (`cloudmanager/data/vm/instances/{vm_public_id}/postgres`, one `vm` param).
