# Marketplace UI — storefront, builder, provision visibility (copper-kiosk)

## What this workflow does
Surfaces the Marketplace in cloud-manager-web: browse published blueprints and create a VM from one
(host + name + optional overrides → instantiate → watch the VM come up on its detail page), compose
and publish blueprints in a builder page (image + ordered playbooks + vars), and show blueprint
provenance plus live ordered-run-chain progress on the VM detail page. One explicitly-scoped phase
assesses whether vorch/porch status writebacks suffice for the UI and delivers findings (plus a
`vorch@cloud-manager` follow-up plan draft only if gaps exist).

## Read before starting
- `deck.md` — scope and pre-written hv MCP calls
- `Orchestrate/ORCH.md` — full task list and execution instructions
- `../../common/toolkit/hive-deck.md` — hv MCP call patterns
- `PLAN.md` — routes, page designs, plumbing, assessment criteria, open-question defaults
- `../lucky-engineblock/PLAN.md` — the backend contract this UI consumes (must be executed first)

## Constraints
- **Series order**: do not start until plan 1 (lucky-engineblock) is executed and merged — the
  endpoints this UI calls must exist.
- Code changes in cloud-manager-web ONLY. No api/mcp/vorch/porch changes; Phase 0005 produces
  findings + a follow-up plan draft, never inline changes.
- Existing Create VM wizard and all /ansible/* pages must remain byte-identical (regression-check
  in Phase 0006).
- Everything new is gated on the `marketplace` feature flag (flag off = zero visible change).
- Follow existing conventions: camelCase wire calls in api.ts, Redux Toolkit slice patterns
  (vmsSlice exemplar), SignalR via the existing hook, SASS + theme system.
- Keep task/test count proportional per phase.
