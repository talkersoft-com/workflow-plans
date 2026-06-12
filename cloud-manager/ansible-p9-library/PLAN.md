# Plan: Library UX — browser, graph, composer, preview, guardrails (ansible-p9-library)

> Plan 9 of 9 in the Ansible Epic series (windward-steamboat). Conforms to
> `master-plans/ANSIBLE-EPIC-CONTRACTS.md`. Workflow: `feature@cloud-manager`.
> Satisfies epic §3 (roles as library objects, composition UX, impact-aware editing), §9
> (library browser, playbook composer, effective-config preview, secret indicators, run
> console) and §10 (guardrails, confirmation, used-by warnings, lint gates). Terminal plan —
> exposes no series contracts. **Depends on: P2, P5, P8** (consumes **C-GRAPH**, **C-RES**,
> **C-DECO**; transitively C-VAR and C-ED).

## Objective

Turn the epic's data layer into the reuse-first UX it exists for: a library browser with
search, tags, and a dependency-graph view over P2's edges; a role-interface composer where
picking roles surfaces their required parameters as a validated form (C-VAR kinds); a
per-target effective-config preview pane (C-RES) before any run; guardrails that make shared
entities safe to edit ("used by N playbooks" warnings, delete blocks, lint-gate toggle); and a
run console that tail-follows P6's log chunks. This plan declares an **internal split**
(Part A: metadata + browser/graph; Part B: composer + preview + guardrails + console).

## Background

- Web today: `/ansible/*` pages are functional CRUD lists; `@rjsf/core` is already a dependency
  (used read-only for playbook argument schemas) — the composer reuses it for required-var
  forms instead of a custom form engine. SignalR is wired for live updates; `RunDetailPage`
  polls today.
- `AnsibleRole` carries only name/description. Epic §3 calls for library metadata: tags,
  supported platforms, maintainer, version. That is a small **additive API change** owned by
  this plan (fields + migration on `ansible_roles`, DTOs, list filtering) — the only non-web
  diff in the plan.
- Required/default/internal classification per role (the "interface") already exists as C-VAR
  records scoped `role`; the composer renders it, it does not define it.

## Design

### Part A — library metadata, browser, graph

- **Role metadata (additive)**: `Tags string[]`, `SupportedPlatforms string[]`, `Maintainer`,
  `Version` on `AnsibleRole` (+ DTOs, PATCH support, list filters `?tag=&platform=&q=`).
  Prefix/entity unchanged (`ansr`); no new entities (contracts §1 lists none for P9).
- **Library browser** (`/ansible/library`): unified search across roles/playbooks/collections
  (existing list endpoints + new filters), tag chips, platform badges, used-by counts inline
  from C-GRAPH (`GET api/v1/entity/{id}/usedBy` counts).
- **Dependency graph view**: render `GET api/v1/entity/{id}/graph?depth=2` as an interactive
  SVG graph (force layout, ~hundreds of nodes; depth/edge-type filters; click-through to
  entities). Custom lightweight component in keeping with the no-component-library convention —
  evaluated against pulling a graph lib in the open questions.

### Part B — composer, preview, guardrails, console

- **Composer** (`/ansible/compose`): pick roles → ordered list (blueprint-attach interaction
  pattern); for each role, its C-VAR interface renders as a form — `required` fields prominent
  (rjsf schema generated from definitions), `default` collapsed with override affordance,
  `internal` hidden; secret-classified params use C-DECO's `<SecretValue>`/ref-picker (writes
  `$secretRef` only). Output: a playbook through existing create endpoints, vars through P4's
  typed forms. Validation before save: every required param satisfied or explicitly deferred to
  inventory/runtime layers (gap-aware via C-RES dry resolve).
- **Effective-config preview pane**: on composer and run pages, per-target pane (inventory +
  host pickers, P8's context bar reused) listing resolved vars + provenance + gaps from C-RES;
  gaps render as blocking chips when `enforceRequired` is on.
- **Guardrails** (epic §3/§10):
  - Edit/delete of any shared entity (role, var, group var, playbook used by blueprints) first
    calls used-by; non-zero → warning modal "used by N playbooks / M tasks" with the list
    (C-GRAPH), destructive confirm in the existing `ConfirmModal` danger pattern.
  - Role-interface breaking-change warning: removing/renaming a `required` var of a role with
    consumers → explicit breakage list before save.
  - Lint gate toggle (per P6's open question resolution): a run-page setting that blocks
    trigger while the head revision has lint errors (surface from `plint`).
- **Run console**: `RunDetailPage` upgraded to tail-follow `GET …/log/chunk?afterSeq=` (poll;
  SignalR push as stretch), per-task grouping from chunk content, status/progress per target,
  `mode: "check"` runs labelled "dry-run — would change".

**Before** (today — reuse is invisible; delete is blind):
```
DELETE role → 204. Whatever referenced it is now broken.
```
**After** (C-GRAPH-backed guardrail):
```
Delete "nginx-base"?  ⚠ Used by 3 playbooks, 1 role (depends_on):
  pb_SiteDeploy, pb_EdgeCache, pb_Canary, ansr_TlsBase
[Cancel] [Delete anyway — breaks 4 consumers]
```

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000 | Setup | hv_status + hv_next; record branch in deck.md |
| 0001 | A: role metadata | Additive `ansible_roles` fields + migration, DTOs, list filters (only non-web diff) |
| 0002 | A: browser + graph | `/ansible/library` page, search/tags/badges, used-by counts, graph view component |
| 0003 | B: composer | Role picking, C-VAR interface forms (rjsf), secret ref-picker, validate-before-save |
| 0004 | B: preview + guardrails | Effective-config pane (C-RES), used-by warning modals, breaking-change check, lint-gate toggle |
| 0005 | B: run console + regression | Chunk tail-follow, check-mode labelling; existing pages byte-identical; e2e composer→preview→run |

## Open questions

1. **Graph rendering**: hand-rolled SVG force layout vs a small dependency (e.g. d3-force
   only)? Default: d3-force for layout math only, custom React rendering — no component-library
   exception.
2. **Composer output target**: always a new playbook, or also "recompose existing"? Default:
   new playbook only this plan; recompose-existing needs P3 structured edits end-to-end and is
   an explicit deferral.
3. **Library unification depth**: include inventories (P1) in the browser? Default: yes for
   search results, no dedicated inventory management UX yet — full inventory pages are a
   recorded deferral (epic coverage map).
4. **SignalR for chunks**: extend the hub now or poll `afterSeq`? Default: poll (2s) this plan;
   hub push as follow-up if console feel demands it.
