# Ansible Epic — Shared Contracts (windward-steamboat)

Every plan in the series (P1–P9) conforms to this file. It is the single source of truth for new
entity names, public-id prefixes, schema namespaces, route shapes, MCP tool names, event types,
and the three cross-consumed interface contracts. A plan that needs to deviate must change THIS
file in its own PR and state the change in its Design section — never drift silently.

Researched against main as of 2026-06-11: `EntityPrefixRegistry.cs` (38 registered prefixes),
controller routes (`api/v1/<resource>`, singular), the 86-tool MCP (`cloud_<noun>_<verb>`),
porch message structs in vorch-lib (`models/messages/`), and the marketplace/blueprint series
(shipped: `bp`/`bpp`/`bpev`, sequencer, `marketplace` schema).

---

## 1. New entities — names, prefixes, schemas, owners

Collision-checked against `EntityPrefixRegistry.cs` on main, which registers exactly:
`vm snap vmi host dhost org usr team role iso naddr ncfg ocmd vdev urole uteam trole ff pb asgn
run pbrev ansr rf rfrev pgr plr plrf plrfr prt acoll hacoll acrun pcreq vmev bp bpp bpev`.
No prefix below appears in that list, and none duplicates another row below.

| Entity | Prefix | Table (schema) | Owning plan | Notes |
|---|---|---|---|---|
| Inventory | `inv` | `inventories` (`ansible`) | P1 | named inventory; soft-delete (DeletedAt) |
| InventoryGroup | `invg` | `inventory_groups` (`ansible`) | P1 | self-FK ParentGroupId for child groups |
| InventoryHost | `invh` | `inventory_hosts` (`ansible`) | P1 | optional FK → VirtualMachine; or external host |
| InventoryGroupMembership | `igm` | `inventory_group_memberships` (`ansible`) | P1 | host↔group many-to-many |
| GroupVar | `gvar` | `group_vars` (`ansible`) | P1 | value jsonb OR SecretRef FK (P4 retrofits typing) |
| HostVar | `hvar` | `host_vars` (`ansible`) | P1 | same value rules as GroupVar |
| InventoryEvent | `invev` | `inventory_events` (`ansible`) | P1 | append-only, VmEvent pattern |
| PlaybookPlay | `play` | `playbook_plays` (`ansible`) | P2 | ordered within playbook revision |
| PlaybookTask | `ptask` | `playbook_tasks` (`ansible`) | P2 | ordered within play/handler/role |
| PlaybookHandler | `hdlr` | `playbook_handlers` (`ansible`) | P2 | |
| VarBlock | `vblk` | `var_blocks` (`ansible`) | P2 | structural vars: block (play vars, role defaults/vars) |
| EntityEdge | `edge` | `entity_edges` (`ansible`) | P2 | typed graph edge, see §5 |
| YamlDocument | `ydoc` | `yaml_documents` (`ansible`) | P3 | canonical source text + decomposition map per revision |
| VariableDefinition | `vdef` | `variable_definitions` (`ansible`) | P4 | the registry record, see §7 |
| SecretRef | `sref` | `secret_refs` (`ansible`) | P4 | vault pointer; never a value |
| VariableEvent | `varev` | `variable_events` (`ansible`) | P4 | append-only |
| ResolutionSnapshot | `rsnap` | `resolution_snapshots` (`vm`) | P5 | persisted effective-value report per run |
| PlaybookLintResult | `plint` | `playbook_lint_results` (`vm`) | P6 | per playbook revision; jsonb findings |
| PlaybookRunLogChunk | `logc` | `playbook_run_log_chunks` (`vm`) | P6 | ordered chunks per run target |

P7, P8, P9 introduce **no entities**. P7/P8 are web-only; P9's only API diff is additive
metadata fields on the existing `AnsibleRole` (tags/platforms/maintainer/version — no new
prefix, no registry change).

Rules every entity follows: Audit base (`PublicId`, `Created`, `CreatedBy`, `Modified`,
`ModifiedBy`), snake_case DB columns, camelCase wire JSON, public-id registered in
`EntityPrefixRegistry`, soft-delete only where the row is user-facing and referenced
(`Inventory` only, filtered unique index on name like `Blueprint`).

## 2. Schema namespaces

- `ansible` schema: all new authoring/data entities (P1–P4). Collections already live here.
- `vm` schema: run-adjacent entities (P5 `resolution_snapshots`, P6 `playbook_lint_results`,
  `playbook_run_log_chunks`) — next to `playbook_runs`/`playbook_run_targets`.
- No new Postgres schemas are created by this epic.

## 3. API route shapes

Base: `api/v1/<resource>` singular, nested one level per aggregate, async ops return
**202 + `{ "runId": ... }`**, feature-flag gated via `[RequireFeatureFlag(...)]`.

| Area | Routes | Owner |
|---|---|---|
| Inventory | `api/v1/inventory` CRUD; `…/{id}/group` (+ `/{gid}/var`); `…/{id}/host` (+ `/{hid}/var`); `…/{id}/group/{gid}/member` | P1 |
| Decomposition | `api/v1/playbook/{pid}/play` (+ `/{playId}/task`), `…/handler`, `…/varBlock` — read + structured edit | P2 |
| Graph | `api/v1/entity/{publicId}/usedBy` (reverse index), `api/v1/entity/{publicId}/graph?depth=` | P2 |
| Round-trip | `api/v1/roundtrip/decompose`, `api/v1/roundtrip/recompose` (service-internal; auth-scoped) | P3 |
| Variables | `api/v1/variable` CRUD; `api/v1/variable/{id}/secretRef` PUT/DELETE | P4 |
| Resolution | `POST api/v1/resolution/preview` (§8) | P5 |
| Lint / check | `POST api/v1/playbook/{pid}/lint` → 202+runId (worker reports via `POST …/lint/{runId}/result`); `POST api/v1/playbook/{pid}/run` body gains optional `"mode": "apply"\|"check"` (additive, default `apply`) | P6 |
| Log chunks | ingest `POST api/v1/run/{runId}/log/chunk` (worker auth); read `GET api/v1/run/{runId}/log/chunk?afterSeq=&max=` | P6 |

## 4. MCP tool names

Pattern stays `cloud_<noun>_<verb>`, ids via `idSchema`, camelCase bodies, read/mutate separation,
runId polling for async. New tools (registered in profile `hv.cloud`):

| Tools | Module | Owner |
|---|---|---|
| `cloud_inventory_list/get/create/update/delete`, `cloud_inventory_group_create/update/delete`, `cloud_inventory_host_attach/detach`, `cloud_group_var_set/list/delete`, `cloud_host_var_set/list/delete` | `inventory.ts` | P1 |
| `cloud_entity_used_by`, `cloud_entity_graph`, `cloud_playbook_play_list`, `cloud_playbook_task_list` | `graph.ts` | P2 |
| `cloud_variable_list/get/create/update/delete`, `cloud_variable_secret_ref_set` | `variables.ts` | P4 |
| `cloud_resolution_preview` | `resolution.ts` | P5 |
| `cloud_playbook_lint`, `cloud_playbook_check`, `cloud_run_log_chunks` | extends `ansible.ts` | P6 |

## 5. Edge graph contract (P2 exposes; P9 consumes)

`EntityEdge`: `{ fromPublicId, fromType, toPublicId, toType, edgeType }` with `edgeType` enum
(snake_case on the wire): `depends_on`, `includes`, `consumed_by`, `targets`, `defines`,
`overrides`. Unique on (from, to, edgeType). Used-by = incoming edges. Cycle detection on
`depends_on` insert (400 on cycle). Reverse-index consistency is an invariant: edits that change
references rewrite edges in the same transaction. Before P4 exists, variable nodes use
`var:<name>` pseudo-ids in the from/to slots; P4's migration re-keys matching names to
`vdef_…` ids in place.

## 6. Event types (append-only pattern)

Per-aggregate event tables follow VmEvent/BlueprintEvent exactly: `EventType` (max 64,
snake_case), `Payload` jsonb, `Actor`, `OccurredAt`, Audit base, never updated.

- `inventory_events` (P1): `created`, `updated`, `deleted`, `group_added`, `group_removed`,
  `host_attached`, `host_detached`, `var_set`, `var_deleted`
- `variable_events` (P4): `created`, `updated`, `deleted`, `classified_secret`,
  `secret_ref_set`, `secret_ref_removed`
- New `vm_events` types (additive): `playbook_check_completed` (P6), `playbook_lint_completed` (P6)

## 7. Variable / secret typing contract — **C-VAR** (P4 exposes; P5, P8 consume)

`VariableDefinition` wire shape (camelCase):

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

Invariants (P5 and P8 may rely on these without re-checking):
- `sensitivity: "secret"` ⇒ `defaultValue` is **null** and `secretRef` is non-null. The DB never
  holds a secret value — refs only (epic §4; existing `SecretInputRef` pattern in vorch-lib).
- `kind: "required"` ⇒ resolution reports a gap when no winner exists (§8).
- `name` + `scope` + `ownerPublicId` unique.

**Before** (today, untyped — `VmPlaybookAssignment.VarsOverride` jsonb):
```json
{ "api_token": "hvs.plaintext-oops", "hostname": "web-01" }
```
**After** (P4 — the token is a typed registry entry; the override holds a ref, never the value):
```json
{ "api_token": { "$secretRef": "sref_XyZ" }, "hostname": "web-01" }
```

## 8. Resolution API contract — **C-RES** (P5 exposes; P8, P9 consume)

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

Invariants:
- `effectiveValue` for `sensitivity: "secret"` is **always** the literal mask `"***"` — the API
  never returns secret cleartext; provenance still names the winning `sref_…`.
- `precedenceLevel` follows Ansible's documented precedence order (P5 ships the table as data).
- Always scoped to an operation context (playbook + inventory + host) — never abstract.

**Before** (today): vars are merged blind inside porch (`extravars` file); no provenance, no
preview, gaps surface only as ansible runtime failures.
**After** (P5): the same merge is computed API-side ahead of run, with winner + overridden chain
+ gap list; porch behavior is unchanged (it receives the merged result exactly as today).

## 9. Editor component contract — **C-ED** (P7 exposes; P8 consumes)

P7 replaces the `<textarea>`-based `RoleFileEditor` with CodeMirror 6 behind the **same consumer
API** (drop-in: `initialContent`, `onSave`, `onDirtyChange` keep working), and adds extension
points P8 plugs into:

```tsx
<AnsibleYamlEditor
  initialContent={string}
  onSave={(content, changeSummary?) => Promise<void>}
  onDirtyChange={(dirty) => void}
  fragmentMode={boolean}                  // tolerate non-document YAML chunks
  decorations={EditorDecoration[]}        // P8 supplies; P7 renders
  diagnostics={EditorDiagnostic[]}        // lint findings → squiggles + gutter
  hoverProvider={(varName, pos) => Promise<HoverContent | null>}  // P8 wires to C-RES
/>

type EditorDecoration = { from: number; to: number; kind: "secret" | "required" | "vaultRef"; varName: string }
type EditorDiagnostic  = { from: number; to: number; severity: "error" | "warning" | "info"; message: string; rule?: string }
```

**Before** (today, `RoleFileEditor.tsx`): plain `<textarea>`, monospace CSS, no highlighting.
**After** (P7): CM6 with YAML + embedded Jinja2 grammar; `{{ api_token }}` is a template token,
decoratable by kind — P8 makes it render as a secret badge with hover provenance.

## 10. Inter-plan dependency table

| # | Plan | Depends on | Consumes | Exposes |
|---|---|---|---|---|
| P1 | ansible-p1-inventory | — | conventions (§1–§3) | inventory entities + APIs (**C-INV**) |
| P2 | ansible-p2-decomposition | P1 | C-INV | decomposition records, edge graph, used-by APIs (**C-GRAPH**, §5) |
| P3 | ansible-p3-roundtrip | P2 | C-GRAPH | round-trip service (**C-RT**) |
| P4 | ansible-p4-variables | P1, P2 | C-INV, C-GRAPH | typing contract (**C-VAR**, §7) |
| P5 | ansible-p5-resolution | P1, P2, P4 | C-INV, C-GRAPH, C-VAR | resolution API (**C-RES**, §8) |
| P6 | ansible-p6-porch | — | porch message contract (additive only) | check/lint/log-stream (**C-PORCH**, §3/§6) |
| P7 | ansible-p7-editor | — | — | editor component (**C-ED**, §9) |
| P8 | ansible-p8-decorations | P4, P5, P7 | C-VAR, C-RES, C-ED | decoration layer (**C-DECO**) |
| P9 | ansible-p9-library | P2, P5, P8 | C-GRAPH, C-RES, C-DECO (transitively C-VAR, C-ED) | library UX (terminal) |

Execution order (ledger order): P1 → P2 → P3 → P4 → P5 → P6 → P7 → P8 → P9.
P6 and P7 have no data-series dependencies and could in principle run early, but the series
executes serially in ledger order to keep one-plan-in-flight review discipline.

## 11. Conventions restated (binding on all nine)

- **Audit base** on every entity; `PublicId` max 32; prefixes registered in
  `EntityPrefixRegistry` (collision-check at PR time against §1).
- **camelCase wire JSON** (MCP client rejects snake_case keys); **snake_case DB columns**;
  public ids on the wire, never raw UUIDs.
- **Soft-delete** only where listed in §1, via `DeletedAt` + `HasQueryFilter` + filtered unique
  indexes (`Blueprint` pattern).
- **Append-only events** per aggregate (§6); recorded via an `I<Aggregate>EventService` like
  `IVmEventService.RecordEventAsync`.
- **Feature flag — decision: ONE epic flag.** Key `ansible-studio` (first multi-word key;
  Key is varchar(64), hyphens fine). It gates every new API surface and web route in P1–P9 via
  `[RequireFeatureFlag("ansible-studio")]` / `FeatureGate`. The existing `playbooks` flag and its
  porch health probe are untouched; P6's new capabilities PATCH health for `ansible-studio`
  through the existing `…/FeatureFlags/{key}/health` route shape.
- **No-plaintext-secrets invariant** (epic §4/§11): the DB and all API responses carry secret
  *references* (`sref_…`, vault paths) and masked values only; redaction in porch
  (`redact.ScrubString`) stays in place; the existing intentional MCP exclusion of
  `GET /run/{runId}/secret` stays excluded.
- **Porch message contracts are additive-only** (P6): existing fields of `PlaybookRunMessage`,
  `MaterializedPlaybook`, `MaterializedFile`, `SecretInputRef`, existing queue names, and the
  five run-status strings never change; new fields are optional with `omitempty`; new
  capabilities arrive as NEW structs + NEW queues (e.g. P6's `PlaybookLintMessage` on
  `playbook-lints`), mirroring the collection-install pattern.
- **Async pattern**: long work returns 202 + runId; status via existing run/target polling.
