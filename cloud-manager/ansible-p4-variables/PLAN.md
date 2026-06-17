# Plan: Variable registry — typing, secrets as refs, required enforcement (ansible-p4-variables)

> Plan 4 of 9 in the Ansible Epic series (windward-steamboat). Conforms to
> `master-plans/ANSIBLE-EPIC-CONTRACTS.md`. Workflow: `feature@cloud-manager`.
> Satisfies epic §4 (variable registry, sensitive-parameter typing, no plaintext secrets,
> required-var enforcement). Exposes contract **C-VAR** (contracts §7) — consumed by P5 and P8.
> Depends on: P1 (host/group vars get typed), P2 (consumed-by edges re-key to registry records).

## Objective

Make variables first-class registry records with explicit typing: every variable a role,
playbook, or inventory declares becomes a `VariableDefinition` with a `kind`
(required/default/internal — the role-interface split of epic §3) and a `sensitivity`
(plain/secret). Secret variables never carry values — they carry a `SecretRef` pointing into
Vault, and every override surface (`varsOverride` on assignments and blueprint playbooks)
accepts the `$secretRef` form instead of cleartext. Required variables become enforceable:
the API can answer "what's missing?" before anything runs.

## Background

- Today: vars are opaque jsonb everywhere (`VmPlaybookAssignment.VarsOverride`,
  `BlueprintPlaybook.VarsOverride`, P1's `gvar`/`hvar` values). Nothing distinguishes
  `api_token` from `hostname`; nothing stops a cleartext token landing in the DB.
- Existing secret machinery to build on, not replace: `MaterializedPlaybook.SecretInputs`
  (`[]SecretInputRef{ExtravarName, VaultPath}`) — porch already resolves Vault paths into
  extravars with the worker token at materialization time. P4 routes typed secrets through this
  exact mechanism, so **porch does not change**.
- The no-plaintext-secrets invariant (epic §4/§11, contracts §11) starts being machine-enforced
  here.

## Design

### Data model (schema `ansible`, contracts §1)

| Entity | Prefix | Notes |
|---|---|---|
| VariableDefinition | `vdef` | Name; Scope `global\|playbook\|role\|inventory`; OwnerPublicId nullable; Kind `required\|default\|internal`; Sensitivity `plain\|secret`; DefaultValue jsonb nullable; Description. Unique (name, scope, owner). |
| SecretRef | `sref` | FK VariableDefinition (Cascade); VaultPath (the API-form `cloudmanager/data/…` string porch already understands) |
| VariableEvent | `varev` | append-only; types per contracts §6 (`created`, `updated`, `deleted`, `classified_secret`, `secret_ref_set`, `secret_ref_removed`) |

DB-level invariant (CHECK + service validation): `sensitivity = 'secret'` ⇒ `default_value IS
NULL`; secret values exist only in Vault. C-VAR wire shape verbatim from contracts §7:

```json
{
  "publicId": "vdef_AbC123",
  "name": "api_token",
  "scope": "global | playbook | role | inventory",
  "ownerPublicId": "ansr_… | pb_… | inv_… | null",
  "kind": "required | default | internal",
  "sensitivity": "plain | secret",
  "secretRef": { "publicId": "sref_XyZ", "vaultPath": "cloudmanager/data/…" },
  "defaultValue": { "json": "any" },
  "description": "…"
}
```

### Typed overrides — the `$secretRef` form

Every jsonb override surface accepts `{ "$secretRef": "sref_…" }` as a value. Validation layer
(shared service) rejects writes where a value is a *string assigned to a secret-classified
variable name* — the write must use a ref (409 with guidance). At materialization, the API
expands `$secretRef` values into `SecretInputs` entries on the existing
`GET /playbook/{id}/materialized` response — porch resolves them exactly as it does today.

**Before** (today, contracts §7 example — cleartext lands in the DB):
```json
{ "api_token": "hvs.plaintext-oops", "hostname": "web-01" }
```
**After** (P4 — ref only; Vault holds the value; porch behavior unchanged):
```json
{ "api_token": { "$secretRef": "sref_XyZ" }, "hostname": "web-01" }
```

### Registry population

- Manual CRUD: `api/v1/variable` (+ `/{id}/secretRef` PUT/DELETE), gated `ansible-studio`.
- Harvest assist: P2's decomposer already finds `defines`/`consumed_by` var names; a
  `POST api/v1/variable/harvest?owner=…` endpoint proposes unregistered names (kind defaulted
  `internal`) for one-click registration. Harvest never auto-classifies sensitivity.
- P2 re-key: migration converts `var:` pseudo-id edges to `vdef_…` ids where names match a
  registered definition; unmatched names stay pseudo (still resolvable by name).

### Required enforcement (epic §4)

`GET api/v1/playbook/{pid}/requiredVars?inventoryId=&hostId=` — required `vdef`s for the
playbook's role closure (via P2 `includes`/`depends_on` edges) minus those satisfied by any
override/var layer; returns the missing list. Run trigger (`POST …/run`) gains optional
`"enforceRequired": true` (additive, default false this plan — flipping the default is a P5/P6
open question once gap detection is fully wired).

### MCP (contracts §4)

New module `variables.ts`: `cloud_variable_list/get/create/update/delete`,
`cloud_variable_secret_ref_set`. Read/mutate separation; ids via `idSchema`. The existing
intentional exclusion of `GET /run/{runId}/secret` from MCP stays.

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000 | Setup | hv_status + hv_next; record branch in deck.md |
| 0001 | Entities + migration | `vdef`/`sref`/`varev`, CHECK constraints, P1 var-table nullable `variable_definition_id` retrofit, edge re-key |
| 0002 | Registry API | CRUD + secretRef routes, events, harvest endpoint |
| 0003 | Typed overrides | `$secretRef` validation on every override write path; materialization expands refs into SecretInputs |
| 0004 | Required enforcement | requiredVars endpoint over the role closure; optional `enforceRequired` on run trigger |
| 0005 | MCP + regression | `variables.ts`; cleartext-rejection tests; existing runs (incl. blueprint sequencer) byte-identical when no typed vars exist |

## Open questions

1. **Vault path authoring**: free-text VaultPath on `sref` create, or picker backed by a vault
   list API? Default: free-text now (validated prefix `cloudmanager/`); picker is P9 UX.
2. **Retroactive secret scan**: scan existing `varsOverride` blobs for token-looking strings
   and flag them? Default: report-only admin endpoint, no auto-migration of values.
3. **Classification authority**: who may flip `sensitivity` to `secret` (it's one-way by
   default — declassification requires explicit force flag)? Default: one-way + force flag,
   both evented.
