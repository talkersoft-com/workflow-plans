# Marketplace — VM Blueprints

## What this workflow does
Adds the Blueprint/Marketplace layer above the existing VM+playbook model: Blueprint = base image
(`vmi_`) + ordered playbooks + default specs; Published blueprints form the Marketplace; instantiate
stamps out a VM from the recipe (image + defaults + copied assignments), provisions it through the
existing orchestration path, and (Phase 0003) auto-runs the playbooks in blueprint order. The
existing admin flows — Create-VM wizard and direct playbook attach/apply — keep working
byte-identically.

## Read before starting
- `deck.md` — which deck and repos are in scope; pre-written hv MCP calls
- `Orchestrate/ORCH.md` — full task list and execution instructions
- `../../common/toolkit/hive-deck.md` — hv MCP call patterns
- `PLAN.md` — entity tables, endpoint shapes, sequencer design, open-question defaults

## Constraints
- **No vorch-lib / vorch-service / porch changes.** The design composes existing API-side machinery
  (VM create path, IProvisionService, PlaybookRun enqueue) — if a change there seems needed, stop
  and flag it instead.
- Existing direct VM playbook attach/apply must keep working byte-identically — regression-check it
  in Phase 0006.
- Follow repo conventions exactly: Audit base + EntityPrefixRegistry public IDs (`bp`, `bpp`,
  `bpev`), camelCase wire JSON, soft-delete with filtered unique indexes (VirtualMachine pattern),
  append-only events (VmEvent pattern), AutoMapper/service/controller stack (Playbook stack is the
  exemplar), `[RequireFeatureFlag("marketplace")]` on every new endpoint.
- EF migration must be reversible (working Down()).
- Keep task/test count proportional to each phase; the sequencer (0003) is isolated so it can be
  reviewed/reverted independently.
