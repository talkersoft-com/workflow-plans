# Plan: Ansible Epic planning factory — author the nine-plan series (windward-steamboat)

## Objective

Execute once, produce the complete, reviewable plan series for the Ansible Epic
(`hive-deck-pro/planning/workflow-plans/master-plans/ANSIBLE-EPIC.md`): a shared contracts file,
nine fully-formed plan documents (P1–P9, dependency-ordered, each independently executable under
its named workflow), and a series ledger tracking them to completion. Documents only — zero code
changes. Everything ships as ONE plan-stage PR so the human reviews the whole series in a single
pass, then approves and executes plan-by-plan via the standard attestation + headless cadence.

## Background

- The epic spans data model (decomposed entities, graph, round-trip), variables (registry,
  secrets, precedence), resolution (effective values + provenance), MCP additions, porch
  additions (--check/lint/streaming), a real editor (CM6 + Jinja2 + semantic decorations), and
  library UX. Current system state: marketplace/blueprints shipped; Playbook/PlaybookRevision/
  roles/collections/runs/blueprints as on main; async runId pattern, audit events, and the
  86-tool MCP already match epic §6/§10 closely. The remaining ~⅔ is data-and-semantics work.
- Nine inter-dependent plans written ad hoc would drift on contracts. Hence: contracts file
  first, plans second, cross-consistency check last — all inside one orchestrated run.

## Design

Execution tasks (each maps to a phase):

1. **Contracts first** — `workflow-plans/master-plans/ANSIBLE-EPIC-CONTRACTS.md`: entity names +
   public-id prefixes + schema namespaces, API route shapes, MCP tool naming, owner plan per
   contract, inter-plan dependency table. Must mirror repo conventions (Audit base, camelCase,
   soft-delete, append-only events, feature flags) — research the code before writing.
2. **Nine plans** — folders `cloud-manager/ansible-p1-inventory` … `ansible-p9-library`, each a
   standard reviewable plan (Objective / Background / Design WITH before-after contract examples /
   proportional phases table / open questions), citing the epic §§ it satisfies and the contracts
   it consumes/exposes:
   | # | Codename | Scope | Workflow |
   |---|---|---|---|
   | P1 | ansible-p1-inventory | inventories/groups/host+group vars as records | feature@cloud-manager |
   | P2 | ansible-p2-decomposition | plays/tasks/handlers/vars records, edge graph, used-by APIs | feature@cloud-manager |
   | P3 | ansible-p3-roundtrip | ruamel round-trip service; sidecar-vs-porch decision REQUIRED | feature@cloud-manager |
   | P4 | ansible-p4-variables | variable registry, secret typing, refs-only, required enforcement | feature@cloud-manager |
   | P5 | ansible-p5-resolution | precedence engine, effective value + provenance, gap detection | feature@cloud-manager |
   | P6 | ansible-p6-porch | --check, ansible-lint, chunked log streaming | vorch@cloud-manager |
   | P7 | ansible-p7-editor | CodeMirror 6, YAML+Jinja2, fragments, inline lint | feature@cloud-manager |
   | P8 | ansible-p8-decorations | secret/required badges, hover provenance, masking | feature@cloud-manager |
   | P9 | ansible-p9-library | browser+graph, interface composer, preview pane, guardrails | feature@cloud-manager |
3. **Ledger** — `workflow-plans/master-plans/ANSIBLE-EPIC-LEDGER.md`: # / codename / epic §§ /
   workflow / depends-on / status, all initialized "planned".
4. **Cross-consistency review** — re-read all nine vs contracts + epic; fix drift; write the
   coverage map (every epic § → owning plan; explicit deferrals listed).

Ship: `stage: "plan"` (plans-repo-only) → ONE PR awaiting human review. The run STOPS after
shipping. Approval of the nine is the human's, then per-plan: hv_plan_approve → scaffold →
headless exec, in ledger order.

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000 | Setup | hv_status + hv_init/hv_next |
| 0001 | Contracts | Research current entities/conventions; write ANSIBLE-EPIC-CONTRACTS.md |
| 0002 | Plans P1–P5 | Backend/data series, dependency-ordered, contracts-conformant |
| 0003 | Plans P6–P9 | Porch + web series |
| 0004 | Ledger + cross-consistency + ship | Ledger; consistency pass + coverage map; hv_ship stage "plan"; STOP |

## Open questions

None — plan content decisions belong to the nine plans themselves (each carries its own open
questions for the reviewer).
