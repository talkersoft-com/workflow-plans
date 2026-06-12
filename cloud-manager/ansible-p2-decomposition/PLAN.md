# Plan: Playbook decomposition + entity graph — plays, tasks, handlers, var blocks, used-by (ansible-p2-decomposition)

> Plan 2 of 9 in the Ansible Epic series (windward-steamboat). Conforms to
> `master-plans/ANSIBLE-EPIC-CONTRACTS.md`. Workflow: `feature@cloud-manager`.
> Satisfies epic §2 (decomposed records, relationship graph, reverse indexes, versioning) and
> §3 (reuse beyond roles, impact-aware editing — data layer). Exposes contract **C-GRAPH**
> (contracts §5). Depends on: P1 (graph edges may reference inventory records).

## Objective

Decompose stored playbooks into addressable records — plays, tasks, handlers, and structural
var blocks — and materialize an explicit, typed relationship graph with reverse ("used-by")
indexes. Today a playbook revision is one YAML blob (`PlaybookRevision.Content`); nothing can
answer "which tasks consume `api_token`?" or "which playbooks include role X?" without parsing
YAML on the fly. After this plan, every save re-derives the record set and edge graph for that
revision, and `usedBy`/`graph` APIs power impact analysis, safe deletion, and P9's library UX.

## Background

- Current state on main: `Playbook` (`pb`) + `PlaybookRevision` (`pbrev`) hold full YAML text;
  roles exist as `AnsibleRole`/`PlaybookRole` with files; `PlaybookGlobalRoleRef` (`pgr`) is the
  only "edge" in the system, and it is bespoke.
- **Design stance — derived read model.** The YAML text remains the source of truth for content
  (epic §2 round-trip fidelity is P3's job). P2's records are a *derived index*, rebuilt
  atomically per revision by a decomposer service using YamlDotNet (already a repo dependency).
  Records are therefore allowed to be lossy about formatting; they are never recomposed into
  YAML by this plan. This is what keeps P2 honest next to P3: P2 = read model, P3 = write path.
- Versioning extension: records carry `PlaybookRevisionId`, so "what changed between revisions"
  becomes record diffing, not text diffing.

## Design

This plan declares an **internal split** (allowed by the series tests): Part A = decomposed
records + decomposer pipeline; Part B = edge graph + used-by APIs. One PR, phased within.

### Part A — decomposed records (schema `ansible`, contracts §1)

| Entity | Prefix | Notes |
|---|---|---|
| PlaybookPlay | `play` | FK PlaybookRevision (Cascade); Position; Name; HostsPattern; raw attrs jsonb |
| PlaybookTask | `ptask` | FK play or handler parent; Position; Name; ModuleName; args jsonb; SourceSpan (start/end line) |
| PlaybookHandler | `hdlr` | FK PlaybookRevision; Name; tasks hang off it |
| VarBlock | `vblk` | FK owner (play / role via owner type+id); Kind: `play_vars` \| `role_defaults` \| `role_vars`; entries jsonb |

`SourceSpan` (start line/col, end line/col against the revision's YAML) is what later lets P8
anchor editor decorations to records.

Decomposer service (`IPlaybookDecomposer`): on playbook/role-file save (i.e., wherever a new
`PlaybookRevision`/`RoleFileRevision` is cut today), parse with YamlDotNet, rebuild that
revision's records + edges in one transaction. Parse failure does not block the save (the blob
is still stored); the revision is marked `decompositionState: failed` and surfaces in the API.

### Part B — edge graph + used-by (contracts §5)

`EntityEdge` (`edge`, table `entity_edges`): `{ fromPublicId, fromType, toPublicId, toType,
edgeType }`, unique on (from, to, edgeType). Edge types (wire snake_case): `depends_on`,
`includes`, `consumed_by`, `targets`, `defines`, `overrides`.

Edge sources written by the decomposer + existing mutation paths:
- playbook → `includes` → role (from `roles:` lists and existing `pgr` attachments)
- role → `depends_on` → role (from `meta/main.yml` dependencies) — **cycle detection on insert,
  400 on cycle** (epic §3)
- var name → `consumed_by` → task (Jinja2 `{{ var }}` scan of task args; name-keyed, resolved to
  `vdef` records once P4 exists — until then edges carry the bare name in `fromPublicId` slot
  prefixed `var:` per contracts §5 note)
- inventory/group/host (P1) → `targets` ← play (from `hosts:` patterns where matchable)
- var block → `defines` / `overrides` → var name

APIs (gated `ansible-studio`):
- `GET api/v1/entity/{publicId}/usedBy` — incoming edges, grouped by edgeType, with consumer
  summaries ("used by 3 playbooks, 7 tasks")
- `GET api/v1/entity/{publicId}/graph?depth=N` — bounded neighborhood for P9's visualization
- `GET api/v1/playbook/{pid}/play` (+ `/{playId}/task`), `…/handler`, `…/varBlock` — read; PATCH
  on task/play raw attrs is **deferred to after P3** (structured edit without round-trip would
  violate fidelity; recorded as explicit deferral in the coverage map)

**Before** (today — impact question requires grepping blobs):
```
SELECT content FROM vm.playbook_revisions;   -- then parse YAML by eye
```
**After** (C-GRAPH):
```json
GET api/v1/entity/ansr_Nginx1/usedBy
{ "usedBy": [
  { "edgeType": "includes", "fromPublicId": "pb_SiteDeploy", "fromType": "playbook" },
  { "edgeType": "depends_on", "fromPublicId": "ansr_TlsBase", "fromType": "ansibleRole" }
], "counts": { "playbooks": 1, "roles": 1 } }
```

### MCP (contracts §4)

New module `graph.ts`: `cloud_entity_used_by`, `cloud_entity_graph`,
`cloud_playbook_play_list`, `cloud_playbook_task_list`.

### Migration + backfill

One-off backfill command decomposes every existing head revision (admin endpoint
`POST api/v1/playbook/decompose-all`, idempotent), so used-by answers are complete on day one.

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000 | Setup | hv_status + hv_next; record branch in deck.md |
| 0001 | A: record entities | `play`/`ptask`/`hdlr`/`vblk` entities, prefixes, migration |
| 0002 | A: decomposer | YamlDotNet decomposer, transactional rebuild per revision, failure marking, save-path hook |
| 0003 | B: edge graph | `edge` entity + migration; edge writes from decomposer + role-dep cycle check |
| 0004 | B: APIs + backfill | usedBy/graph/read endpoints, decompose-all backfill, events on graph rebuild failures |
| 0005 | MCP + regression | `graph.ts`, profile registration; existing flows byte-identical; backfill run against seed data |

## Open questions

1. **Var-name edges before P4**: keep `var:` pseudo-ids until P4 re-keys to `vdef_…`, or defer
   `consumed_by` edges entirely to P4? Default: keep pseudo-ids — P5 wants consumption data even
   for unregistered vars; P4's migration re-keys in place.
2. **Decomposition of role files** (tasks/handlers inside roles): same pipeline this plan, or
   playbook-level only? Default: both — role files already have revisions (`rfrev`), and used-by
   is weak without role task records.
3. **Graph depth cap**: default `depth=2`, max 5? Default: yes (perf, epic §11).
