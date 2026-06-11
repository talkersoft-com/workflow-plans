# Plan: Marketplace UI — storefront, blueprint builder, and provision visibility (copper-kiosk)

> Plan 2 of the marketplace series. Executes AFTER plan 1 (`cloud-manager/lucky-engineblock/` —
> backend: entities, controllers, instantiate, run sequencer, seeds, MCP tools) is approved,
> executed, and merged. Plans in this series orchestrate in creation order.

## Objective

Put the Marketplace in front of users in cloud-manager-web: a storefront page to browse published
blueprints and create a VM from one in three clicks (host, name, go), a builder page where
blueprints are composed (image + ordered playbooks + vars) and published, and provision visibility
on the VM detail page — the source blueprint and the live progress of its ordered run chain. Plus
one explicitly-scoped assessment phase that answers whether vorch/porch need any changes to support
this UI (expected answer: no), delivering a findings doc — and a follow-up plan draft for the
`vorch@cloud-manager` workflow if gaps exist.

## Background

- Plan 1 (lucky-engineblock) delivers the API this UI consumes: `GET /api/v1/marketplace`
  (published blueprints), `POST /api/v1/marketplace/{blueprintId}/instantiate`
  (`{hostId, name, description?, memory?, diskSizeGb?}` → `{virtualMachineId, runPlan}`),
  `BlueprintController` CRUD + lifecycle + ordered playbook management, `VirtualMachine.blueprintId`
  provenance, the `marketplace` feature flag, and the run sequencer that auto-applies playbooks in
  blueprint order after provisioning.
- Existing web conventions (verified in repo): React 18 + Vite, Redux Toolkit slices
  (`src/store/slices/vmsSlice` is the exemplar), api client in `src/services/api.ts`, SignalR live
  updates via `src/hooks/useSignalR.ts` (RunDetailPage streams run logs/status with it), feature
  flags gate nav + routes (the `playbooks` flag pattern), SASS + theme system.
- The existing Create VM wizard (`/create`, host→image→config→review) and all `/ansible/*` pages
  must remain byte-identical — the Marketplace is an additional path, not a replacement.

## Design

### Routes + navigation

| Route | Page | Gate |
|---|---|---|
| `/marketplace` | MarketplacePage — storefront | `marketplace` flag |
| `/blueprints` | BlueprintsPage — list + create | `marketplace` flag |
| `/blueprints/{id}` | BlueprintDetailPage — composer | `marketplace` flag |

Nav: **Marketplace** as a top-level item (peer of Virtual Machines), **Blueprints** adjacent to it.
Both hidden when the `marketplace` flag is off (exact `playbooks`-flag pattern in the app shell).

### MarketplacePage (storefront)

- Cards over `GET /api/v1/marketplace`: name, description, base-image OS badge (from the vmi
  config metadata the listing exposes), playbook count ("3 playbooks" / "image only").
- "Create VM" on a card opens the instantiate flow (modal or inline panel): host select (reuses the
  wizard's host source), VM name (same validation regex as the wizard), memory/disk pre-filled from
  blueprint defaults and editable. Submit → `POST /marketplace/{id}/instantiate` → navigate to
  `/virtualmachine/{virtualMachineId}` — the user watches their VM come up.

### BlueprintDetailPage (composer)

- Metadata form: name, description, image picker (existing images source), default memory/disk
  (same option sets as the wizard).
- Ordered playbook list: attach from existing playbooks, reorder via up/down position controls
  (no new drag-drop dependency), detach; per-playbook `varsOverride` JSON editor reusing the
  existing assignment vars editor component/pattern.
- Lifecycle: status badge (Draft/Published/Archived) + Publish/Archive actions mapping to
  `POST /blueprint/{id}/publish|archive`; delete only when not Published (matches API rules).

### VM detail: provenance + run-chain progress

- When `vm.blueprintId` is set: "Created from blueprint <name>" with link to the blueprint.
- Run-chain panel: the blueprint's ordered playbooks each showing assignment state
  (pending → queued → running → succeeded/failed) derived from existing run/target endpoints,
  live-updated over the same SignalR mechanism RunDetailPage uses. A failed step shows where the
  chain stopped and links to `/ansible/runs/{runId}` for the full log. No new backend endpoints —
  if composing this from existing endpoints proves impossible, that is a finding for Phase 0005,
  not an inline API change.

### Plumbing

- `src/services/api.ts`: blueprint CRUD/lifecycle/playbook-ordering + marketplace list +
  instantiate functions (camelCase payloads, same error envelope handling as existing calls).
- Redux: `blueprintsSlice` + `marketplaceSlice` (or one combined slice if the team pattern
  prefers), modeled on `vmsSlice`.
- No changes to existing pages beyond the VM detail additions and nav registration.

### Vorch/porch assessment (Phase 0005 — explicitly scoped, per the feature workflow's exception clause)

Question to answer with evidence: is the existing status surface sufficient for this UI?
- Orchestration status writeback (Initialized → InProgress → Completed/Failed) + IP writeback
  granularity for the "watching my VM come up" experience.
- Run/target status + SignalR coverage for the run-chain panel.
Deliverable: `FINDINGS-vorch-status-surface.md` in this plan folder. If gaps require
vorch-lib/vorch-service/porch changes (e.g. additive provision-progress events), the deliverable
includes a drafted follow-up plan targeting the **`vorch@cloud-manager`** workflow — changes are
NOT made inline. Only trivially additive message-field changes may be proposed for explicit
phasing here, and only with the reviewer's (your) sign-off on the findings.

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000 | Setup | hv_status + hv_init/hv_next |
| 0001 | Plumbing | api.ts endpoints, Redux slices, `marketplace` feature-flag wiring, routes + nav (gated, empty pages) |
| 0002 | Marketplace storefront | Cards + instantiate flow + redirect to VM detail |
| 0003 | Blueprint builder | Compose/reorder/vars/publish-archive pages |
| 0004 | VM provenance + run-chain panel | Blueprint link + live ordered-run progress via SignalR |
| 0005 | Vorch/porch status-surface assessment | Findings doc; follow-up `vorch@cloud-manager` plan draft only if gaps |
| 0006 | E2E verification + results | Flag off → nothing visible; flag on → browse jammy + postgres-jammy, instantiate, watch chain to Succeeded; wizard + ansible pages regression; RESULT.md + ship |

## Open questions

1. Instantiate UX: modal on the card vs dedicated `/marketplace/{id}/create` page. Default: modal —
   fewer routes, matches "three clicks".
2. Should the storefront card expose playbook names (transparency) or just the count (simplicity)?
   Default: count on the card, names in an expandable detail.
3. Reorder UX: up/down buttons (default — zero new dependencies) vs drag-and-drop (needs a lib).
