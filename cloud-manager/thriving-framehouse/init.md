# Cloud Manager — Secret-binding verification fixes (MCP param guard + persisted resolved path)

## What this workflow does
Workflow **#5**: fixes the two verification-layer bugs found in the live end-to-end test. (#2) The
MCP `cloud_marketplace_provision` camelCase guard wrongly rejects `secretBindingParams[].params`
map keys (placeholder names like `vm_public_id`), making templated bindings unusable via MCP. (#3)
`GET /api/v1/vm/{publicId}/secret-binding` reports the wrong `resolvedVaultPath` for cross-VM
bindings because the usage row never stored the real path — it recomputes with the listed VM's own
id. Fix: exempt the param-map keys from the MCP guard, and persist + return the actual resolved
path. The feature itself (resolution + injection) is correct and hash-verified — do not change it.

## Read before starting
- `deck.md` — repos in scope; the post-merge **local mcp rebuild** note
- `Execution/Exec.md` — full task list and execution instructions
- `PLAN.md` — both bugs, the fixes, phases, open question
- `../../common/toolkit/hive-deck.md` — hv MCP call patterns

## Constraints
- **Do not change resolution/injection logic** — it is correct (hash-verified end to end).
- **#2 guard fix is narrow:** exempt only the inner `secretBindingParams[].params` map keys; the
  camelCase guard must still apply to all structural/wire field names.
- **#3:** one **reversible** migration (clean up/down) adding `resolved_vault_path`; populate at
  provision from the resolver-computed path; return the stored value, recompute only when null.
- Secret values NEVER appear in output — `resolvedVaultPath` is a path only.
- camelCase JSON; public_ids only; soft-delete authoritative.
- Backward compatible (old null rows → best-effort recompute; valid MCP calls stay valid).
- Migration applied before api restart on deploy. **NO vorch/web changes.**
- After merge: rebuild the **local** cloud-manager-mcp dist + `/mcp` (server deploy doesn't cover
  the dev MCP; `/mcp` alone serves stale dist).

## Out of scope
#4A provisioned-secret registration; host-side seed plaintext scrub; web instantiate VM-picker UI.
