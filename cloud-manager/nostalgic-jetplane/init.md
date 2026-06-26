# Cloud Manager — MCP + API gap-closing for VM secret-binding management

## What this workflow does
Closes the last two MCP/API gaps in the Secret Bindings feature (workflow **#3**), so an agent can
fully manage and **verify** user-defined secrets and bindings: (1) a new API read endpoint
`GET /api/v1/vm/{publicId}/secret-binding` that lists the secret bindings injected into a VM (from
`vm.vm_secret_bindings`, **never the value**), exposed via the MCP tool
`cloud_vm_secret_binding_list`; and (2) the missing `cloud_blueprint_secret_binding_update` MCP tool
wrapping the existing PATCH attachment endpoint. No DB changes, no vorch, no web.

## Read before starting
- `deck.md` — deck + repos in scope; the post-merge `/mcp` reload operator action
- `Execution/Exec.md` — full task list and execution instructions
- `PLAN.md` — the response shape, the no-value rule, phases, open question
- `../../common/toolkit/hive-deck.md` — hv MCP call patterns

## Constraints
- **Secret values NEVER appear** in any endpoint or tool output — only metadata + the Vault *path*
  (same rule that keeps `GET /run/{id}/secret` off the MCP).
- camelCase JSON; **public_ids only** (never leak Guids — `PublicIdByGuidAsync`/`GuidByPublicIdAsync`);
  soft-delete authoritative.
- Mirror the existing `SecretBinding*` controller/service and the playbook attach/reorder MCP tools
  — do not invent a parallel paradigm.
- **NO DB migration; NO vorch changes; NO web changes.**
- Restart cloud-manager-mcp via `/mcp` after merge (new tools) — surface in Results.
