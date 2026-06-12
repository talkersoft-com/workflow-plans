# Ansible inventories as records — Epic plan 1 (ansible-p1-inventory)

## What this workflow does
Adds database-backed Ansible inventories: Inventory/Group/Host records, group membership,
host/group vars as addressable rows, append-only InventoryEvents, CRUD APIs gated by the new
epic-wide `ansible-studio` feature flag, and MCP tools. Pure additive data layer — existing
assignment/run flows keep working byte-identically; P5 later resolves against these records.

## Read before starting
- `deck.md` — repos in scope; pre-written hv MCP calls
- `PLAN.md` — entity tables, route shapes, event types, open-question defaults
- `../../master-plans/ANSIBLE-EPIC-CONTRACTS.md` — prefixes, conventions, C-INV contract
- `../../master-plans/ANSIBLE-EPIC.md` — epic §2, §5

## Constraints
- **Series order**: first executable plan of the epic; requires only main as it exists
  (marketplace/blueprints shipped).
- **Contracts conformance**: entity names/prefixes/schemas exactly per contracts §1; deviations
  must update the contracts file in this plan's PR.
- Code changes in cloud-manager-api and cloud-manager-mcp ONLY. No web, no vorch/porch.
- Follow repo conventions exactly: Audit base + EntityPrefixRegistry, camelCase wire JSON,
  soft-delete with filtered unique index (Inventory only), append-only events, feature-flag
  gating via `[RequireFeatureFlag("ansible-studio")]`.
- Existing playbook/run/blueprint flows must remain byte-identical — regression-check in the
  final phase.
