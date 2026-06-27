# Cloud Manager — Secret Binding provisioning hardening + instantiate param resolution

## What this workflow does
Workflow **#4**: fixes the regression where a blueprint with a templated `{vm_public_id}` secret
binding **bricks its VMs** (resolution throws on the new VM's nonexistent self-path, aborting the
create-vm publish and leaving an orphan), and adds **instantiate-time parameter resolution** so a
templated binding resolves to an **operator-chosen source VM** (e.g. a Postgres server) instead of
the VM being built. **P0** = resolve-before-create + clean 422 (no orphans); **P1** = supply param
values at instantiate. API + MCP only; no DB migration; vorch unchanged.

## Read before starting
- `deck.md` — repos in scope; the post-merge `/mcp` reload note
- `Execution/Exec.md` — full task list and execution instructions
- `PLAN.md` — verified root cause, P0/P1 design, phases, open questions
- `../../common/toolkit/hive-deck.md` — hv MCP call patterns

## Constraints
- **Never brick a VM:** a missing/unresolvable secret must return a clean **422** (binding name +
  Vault **path** only — never the value) with **no orphaned VM record**. Resolve bindings BEFORE
  creating the VM record.
- **Never fall back to the new VM's id** for a `source: vm` param — require the supplied value or 422.
- camelCase JSON; **public_ids only** (never leak Guids); soft-delete authoritative.
- Secret values NEVER appear in any output (errors carry binding name + path only).
- Mirror existing `MarketplaceController` / `SecretBindingResolver` and the marketplace MCP tool.
- **Backward compatible:** no-binding, static-binding, and concrete-path templated bindings keep
  working exactly as today.
- **NO DB migration; NO vorch changes** (the create-vm contract already carries `Secrets` from #2).
- **OUT OF SCOPE:** #4A provisioned-secret registration; host-side seed plaintext scrub; web
  instantiate-form VM-picker UI; cleanup of the existing orphaned `njp-e2e-1` (done operationally).
