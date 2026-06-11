# Plan: Marketplace — VM Blueprints layered over the existing provisioning machinery

## Objective

Add a Blueprint ("recipe") layer above today's VM+playbook model: a Blueprint = one base image
(`vmi_`) + zero-or-more playbooks in an explicit order + default specs. Publishing a Blueprint puts
it in the Marketplace; creating a VM from the Marketplace stamps out the recipe — VM from the image,
playbooks copied to per-VM assignments, provision triggered, runs applied in blueprint order. The
existing admin flow (pick host/image/specs, attach/apply playbooks on any VM) keeps working
byte-identically underneath. End state: an image-only "jammy" blueprint and a "postgres-on-jammy"
blueprint are seeded, browsable in a Marketplace page/MCP tools, and provision end-to-end — the
groundwork for future end-users without any RBAC yet.

## Background

- Today `VmPlaybookAssignment` binds playbooks directly to VMs — `unique(VirtualMachineId,
  PlaybookId)`, **no ordering field** — and VM creation is host+image+specs via
  `POST /api/v1/virtualmachine/create/{hostId}`. Provision goes API → RabbitMQ → vorch; playbook
  runs go assignment → `apply` → PlaybookRun/PlaybookRunTargets → AMQP → porch. All of that stays.
- `VirtualMachineImage` (`vmi_`) already carries OS metadata in Config JSONB — it is the natural
  blueprint ingredient, not the blueprint itself.
- Conventions every new entity must follow (verified in repo): `Audit` base (PublicId via
  `EntityPrefixRegistry` + interceptor, Created/CreatedBy/Modified/ModifiedBy), Stripe-style
  public-id prefixes, camelCase wire JSON (`assertCamelCaseKeys` in MCP), soft-delete via
  `DeletedAt` + global query filter (pattern: VirtualMachine), append-only event log (pattern:
  `VmEvent`), AutoMapper profile + service + controller per entity (exemplar: Playbook/
  PlaybookAssignment stack), feature flags via `[RequireFeatureFlag]`.
- Deck convention: **no vorch-lib / vorch-service / porch changes.** Everything here is API-side
  composition of machinery that already exists.
- Membership entities (Organization/User/Team/Role) are stubs — no RBAC in this plan, but publish
  state is designed so org-scoping can be added later without remodeling.

## Design

### Data model (new schema: `marketplace`)

**Blueprint** (`bp_` prefix; soft-delete like VirtualMachine):
| Field | Type | Notes |
|---|---|---|
| Id | Guid PK | uuid_generate_v4() |
| Name | string(255), required | unique where deleted_at is null |
| Description | string(4096), required | |
| VirtualMachineImageId | Guid FK → VirtualMachineImage, required, Restrict | the base (`vmi_`) |
| DefaultMemory | int, required | MB, same semantics as VirtualMachine.Memory |
| DefaultDiskSizeGb | int, required | 5–500 validation, same as VM |
| Config | JSONB, optional | future default overrides |
| Status | enum BlueprintStatus: Draft \| Published \| Archived | default Draft |
| DeletedAt | DateTime? | soft delete |

**BlueprintPlaybook** (`bpp_` prefix; the ordered join — ordering is the new capability):
| Field | Type | Notes |
|---|---|---|
| Id | Guid PK | |
| BlueprintId | Guid FK → Blueprint, required, Cascade | |
| PlaybookId | Guid FK → Playbook, required, Restrict | |
| Position | int, required | 0-based run order |
| VarsOverride | JSONB, required, default `{}` | copied to the per-VM assignment at instantiate |
Constraints: `unique(BlueprintId, PlaybookId)`, `unique(BlueprintId, Position)`, index (BlueprintId, Position).

**BlueprintEvent** (`bpev_` prefix; append-only, mirrors VmEvent exactly): BlueprintId FK,
EventType (`created|updated|published|archived|playbook_attached|playbook_detached|reordered|instantiated`),
Payload JSONB, Actor, OccurredAt. No soft delete.

**VirtualMachine** (additive change only): nullable `BlueprintId` FK → Blueprint (Restrict) for
provenance — which recipe stamped this VM. Admin-created VMs stay null. No other change; existing
unique indexes/filters untouched.

The Marketplace is not a table — it is the query `Blueprints where Status = Published and
DeletedAt is null`. (If later we need per-listing pricing/visibility, a Listing entity can wrap
Blueprint without remodeling — that is why publish state lives on Blueprint, not on a join.)

### API (cloud-manager-api) — all under `[RequireFeatureFlag("marketplace")]`

`BlueprintController` (`/api/v1/blueprint`), modeled on the Playbook/PlaybookAssignment stack:
- `GET /list` — all non-deleted blueprints (admin/builder view)
- `GET /{id}` — detail incl. ordered playbooks
- `POST /` — create `{name, description, vmi, defaultMemory, defaultDiskSizeGb, config?}`
- `PATCH /{id}` — update mutable fields (Draft/Published both editable; edits to Published log an event)
- `DELETE /{id}` — soft delete (only Draft/Archived; Published must be archived first)
- `POST /{id}/publish`, `POST /{id}/archive` — lifecycle transitions (Draft→Published→Archived;
  Archived→Published allowed; every transition logs a BlueprintEvent)
- `POST /{id}/playbook` `{playbookId, position?, varsOverride?}` (position defaults to end)
- `PATCH /{id}/playbook/{bppId}` `{position?, varsOverride?}` — reorder renumbers atomically
- `DELETE /{id}/playbook/{bppId}`

`MarketplaceController` (`/api/v1/marketplace`):
- `GET /` — published blueprints only (name, description, image summary, playbook count): the storefront
- `POST /{blueprintId}/instantiate` — `{hostId, name, description?, memory?, diskSizeGb?}`
  (overrides win over blueprint defaults). Transactionally: create VirtualMachine (from image +
  defaults, `BlueprintId` set, reusing the exact create path the wizard uses — MAC/IP allocation,
  VmEvent logging), copy each BlueprintPlaybook into a `VmPlaybookAssignment` (VarsOverride
  copied), trigger orchestration provision (existing `IProvisionService` path), log
  `instantiated` BlueprintEvent + VmEvent. Returns `{virtualMachineId, runPlan: [assignmentIds in order]}`.

### Run sequencing after provision (Phase 0003, isolated by design)

MVP (Phase 0002) leaves runs to the existing explicit `apply`. Phase 0003 adds the sequencer:
- The API already consumes orchestration status writebacks (VM → OrchestrationStatus=Completed)
  and playbook-run status updates. Hook those two transitions in a new `BlueprintRunSequencer`
  service: when a VM with `BlueprintId != null` reaches OrchestrationStatus=Completed, enqueue the
  run for the assignment matching the blueprint's Position 0; when a sequenced run reaches
  Succeeded, enqueue the next position. Failure stops the chain and logs a VmEvent
  (`blueprint_run_chain_failed`); the user can fix and `apply` manually from there.
- Ordering state is derived, not stored: VM.BlueprintId → BlueprintPlaybooks ordered by Position,
  matched to the VM's assignments by PlaybookId, next = first whose LastRunStatus != Succeeded.
  No schema change to VmPlaybookAssignment, no new queue infrastructure, no vorch/porch change.

### MCP (cloud-manager-mcp)

New `src/tools/marketplace.ts`, registered in the `hv.cloud` profile:
`cloud_blueprint_list/get/create/update/delete`, `cloud_blueprint_publish/archive`,
`cloud_blueprint_playbook_attach/detach/reorder`, `cloud_marketplace_list`,
`cloud_marketplace_provision` (instantiate). camelCase payloads; same client/error patterns as
`vms.ts`.

### Web (cloud-manager-web)

- `/marketplace` — browse published blueprints (cards: name, base image OS badge, playbook count);
  "Create VM" flow from a card: pick host + name + optional memory/disk overrides (recipe replaces
  the image/spec steps of the wizard), then redirects to the VM detail page ("then the users can
  see the vm").
- `/blueprints` (+ `/blueprints/{id}`) — builder/admin: compose image + ordered playbooks
  (drag-or-buttons reorder), edit vars per playbook, publish/archive.
- Nav: "Marketplace" as a top-level item (peer of Virtual Machines), "Blueprints" under it or
  adjacent; both gated on the `marketplace` feature flag (same pattern as the `playbooks` flag).
- Existing Create VM wizard and all `/ansible/*` pages untouched.

### Migration + seed

- One EF migration: `marketplace.blueprints`, `marketplace.blueprint_playbooks`,
  `marketplace.blueprint_events`, `vm.virtual_machines.blueprint_id` (nullable FK), enum
  `BlueprintStatus`, public-id indexes. Reversible Down().
- Seed (same mechanism as seed/playbooks/): blueprint **jammy** — ubuntu jammy `vmi`, zero
  playbooks (the degenerate case proves itself); blueprint **postgres-jammy** — same image +
  `pg-14-jammy` playbook at position 0. Both seeded as Published so the marketplace demos
  immediately.
- Feature flag row `marketplace` seeded disabled.

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000 | Setup | hv_status + hv_init/hv_next |
| 0001 | Data model + migration + seeds | Entities (Blueprint, BlueprintPlaybook, BlueprintEvent), VM.BlueprintId, prefixes in EntityPrefixRegistry, DbContext config, reversible migration, jammy + postgres-jammy seeds, `marketplace` feature flag |
| 0002 | API: blueprint + marketplace controllers | Services/DTOs/mappers/controllers for CRUD, lifecycle, ordered playbook management, marketplace listing, instantiate (MVP: assignments + provision; runs via existing apply); feature-flag gating; BlueprintEvents |
| 0003 | Run sequencer | BlueprintRunSequencer hooked to orchestration-completed and run-succeeded writebacks; ordered auto-run with stop-on-failure semantics + events |
| 0004 | MCP marketplace tools | `marketplace.ts` tools incl. provision-from-marketplace; profile registration; dist rebuild |
| 0005 | Web: Marketplace + Blueprints pages | Browse/instantiate flow + builder page with ordering; nav + feature flag; existing pages untouched |
| 0006 | End-to-end verification + results | Seeded jammy instantiates from marketplace (no playbooks); postgres-jammy instantiates and reaches a succeeded ordered run on a live VM; direct attach/apply regression check; RESULT.md + ship |

Phases 0002/0003 are deliberately split so the riskiest piece (event-driven sequencing) can be
reviewed and reverted independently of the CRUD surface.

## Open questions

1. **Naming**: "Marketplace" (default, used throughout) vs "Catalog". Rename is cheap before
   Phase 0002 lands, expensive after.
2. **Sequencer trigger source**: hook the AMQP status-writeback consumer directly (synchronous in
   the consumer) vs a lightweight poller on OrchestrationStatus/LastRunStatus. Default: hook the
   consumer — it is where state transitions already materialize; poller only if the consumer path
   proves awkward in Phase 0003.
3. **Published-blueprint mutability**: this plan allows editing Published blueprints (with events).
   Alternative: require re-publish or versioned blueprints (mirroring PlaybookRevision). Default:
   editable now, revisions later if end-users need stability guarantees.
4. Does instantiate let the caller add extra playbooks not in the blueprint? Default: no — keep the
   recipe contract clean; admins can attach directly to the VM afterward (existing flow).
