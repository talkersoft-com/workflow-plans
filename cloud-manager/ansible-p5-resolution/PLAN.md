# Plan: Resolution engine — effective values, provenance, gap detection (ansible-p5-resolution)

> Plan 5 of 9 in the Ansible Epic series (windward-steamboat). Conforms to
> `master-plans/ANSIBLE-EPIC-CONTRACTS.md`. Workflow: `feature@cloud-manager`.
> Satisfies epic §5 (effective-value preview, provenance, operation context, gap detection) and
> §6 partial (MCP resolve tools). Exposes contract **C-RES** (contracts §8) — consumed by P8
> and P9. Depends on: P1 (inventory layers), P2 (consumption graph), P4 (C-VAR typing).

## Objective

Compute, before anything runs, the effective value of every variable in a concrete operation
context — playbook + inventory + host — together with where the winner came from, what it
overrode, and what's missing. Today the merge happens invisibly inside porch's extravars
assembly at execution time; failures surface as ansible runtime errors. After this plan the API
answers "what will `api_token` be on web-01, and why?" with a precedence-ordered provenance
chain, secrets always masked, and a gap list that makes required-var enforcement real.

## Background

- Inputs now exist in records: role defaults / role vars / play vars (P2 `vblk`), inventory
  group/host vars (P1 `gvar`/`hvar` with group nesting), assignment + blueprint overrides
  (jsonb, `$secretRef`-aware per P4), extra-vars at trigger time. P4 typing tells us
  sensitivity and requiredness (C-VAR §7, consumed verbatim).
- Ansible's documented variable precedence (~22 levels) is the semantic contract; only the
  levels representable in our model participate (the table itself records which levels are
  out of scope, e.g. ansible.cfg-level sources porch doesn't use).
- Existing run snapshot: `PlaybookRun.VarsSnapshot` stores the merged map. It stays; this plan
  adds provenance alongside, not a replacement.

## Design

### Precedence engine (cloud-manager-api service, pure + unit-tested)

`IVariableResolutionService.Resolve(context)` where context = playbookId (+revision), inventoryId,
hostId, optional extraVars. The precedence table ships as *data* (ordered list of layer
descriptors mapped to Ansible's documented order); the engine folds layers host-up
(role defaults → inventory group vars by nesting depth → host vars → play vars → assignment
override → blueprint override → extra vars), producing per-variable: winner (layer + source
record public id), overridden chain, state (`resolved | undefined | missingRequired` — required
kinds from C-VAR with no winner are `missingRequired`).

Masking is engine-level, not endpoint-level: any variable whose `vdef` says
`sensitivity: "secret"` (or whose winning value is a `$secretRef`) reports
`effectiveValue: "***"` with the ref's public id in provenance. Cleartext cannot leak through
any caller because the engine never returns it (contracts §8 invariant).

### Preview API — C-RES verbatim (contracts §8)

```
POST api/v1/resolution/preview
{ "playbookId": "pb_…", "inventoryId": "inv_…", "hostId": "invh_…", "extraVars": { … } }
```
→ 200:
```json
{
  "variables": [{
    "name": "api_token",
    "state": "resolved | undefined | missingRequired",
    "sensitivity": "plain | secret",
    "effectiveValue": "***",
    "winner": { "precedenceLevel": 14, "source": "host_vars", "sourcePublicId": "hvar_…" },
    "overridden": [{ "precedenceLevel": 4, "source": "role_defaults", "sourcePublicId": "vblk_…" }]
  }],
  "gaps": [{ "name": "db_password", "kind": "required", "reason": "missingRequired" }]
}
```

Gated `ansible-studio`. Side-effect-free (idempotent read; POST only because the context body
is structured).

**Before** (today — the only visibility is porch's merged extravars file, post-hoc):
```json
{ "secrets_prefix": "cloudmanager/data/playbook-secrets/run_…", "custom_var1": "value1" }
```
**After** (C-RES): the response above — winner, overridden chain, masked secrets, gap list,
all *before* the run exists.

### Run-time snapshot (`rsnap`, schema `vm`, contracts §1)

`ResolutionSnapshot`: FK PlaybookRun; per-target resolution report (jsonb, same C-RES shape,
secrets masked) captured at enqueue. Gives the audit trail "who ran what with which resolved
vars" (epic §10) provenance depth that `VarsSnapshot` lacks. Read via
`GET api/v1/run/{runId}/resolution`.

### Gap detection wiring

- `POST …/run` with P4's `enforceRequired: true` now consults the engine; `missingRequired`
  gaps → 422 with the gap list (flag stays opt-in this plan; default-on is an open question).
- Per-target gap evaluation (epic §5 "per target"): preview accepts `hostId: null` to resolve
  for all hosts of the inventory, returning a per-host gap matrix (bounded; see perf).

### MCP (contracts §4)

New module `resolution.ts`: `cloud_resolution_preview` (read-only). Output mirrors C-RES;
secrets arrive pre-masked from the engine.

### Performance (epic §11)

Layer inputs are record queries, batched per context (no N+1 over hosts); group-nesting flatten
is iterative with the P1 acyclicity guarantee; whole-inventory preview capped (default 200
hosts, 413 beyond) — responsive previews on large inventories are a stated requirement, so the
phase includes a perf test over a synthetic 1k-var / 200-host inventory.

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000 | Setup | hv_status + hv_next; record branch in deck.md |
| 0001 | Precedence engine | Layer descriptors as data, fold algorithm, masking, unit suite incl. C-VAR fixtures |
| 0002 | Preview endpoint | C-RES DTOs verbatim, controller, flag gating, per-host matrix mode |
| 0003 | Run snapshot | `rsnap` entity + migration, capture at enqueue, run resolution read endpoint |
| 0004 | Gap wiring + MCP | enforceRequired 422 path; `resolution.ts`; gap surfacing in run-list DTOs (additive field) |
| 0005 | Perf + regression | Synthetic-scale perf test; existing run flows byte-identical with engine disabled-by-flag |

## Open questions

1. **`enforceRequired` default**: flip to true once P6's check mode lands, or leave opt-in until
   P9's UI can render gaps? Default: leave opt-in; revisit in P9.
2. **Precedence completeness**: model extra layers we don't store (e.g. play `vars_files`) as
   explicit "unmodeled" provenance entries or omit? Default: explicit unmodeled markers — honest
   provenance beats tidy provenance.
3. **rsnap retention**: keep forever like events, or prune with runs? Default: lifecycle-coupled
   to the run row (it is run audit data).
