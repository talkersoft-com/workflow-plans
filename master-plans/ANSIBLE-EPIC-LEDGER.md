# Ansible Epic — Series Ledger (windward-steamboat)

Tracks the nine-plan series for `master-plans/ANSIBLE-EPIC.md`. All plans conform to
`master-plans/ANSIBLE-EPIC-CONTRACTS.md`. Update the status column as each plan moves through
planned → approved → executing → merged.

## Series table

| # | Codename | Epic §§ | Workflow ref | Depends on | Status |
|---|---|---|---|---|---|
| P1 | ansible-p1-inventory | §2 (partial), §5 (dep) | feature@cloud-manager | — | planned |
| P2 | ansible-p2-decomposition | §2, §3 | feature@cloud-manager | P1 | planned |
| P3 | ansible-p3-roundtrip | §2 | feature@cloud-manager | P2 | planned |
| P4 | ansible-p4-variables | §4 | feature@cloud-manager | P1, P2 | planned |
| P5 | ansible-p5-resolution | §5, §6 (partial) | feature@cloud-manager | P1, P2, P4 | planned |
| P6 | ansible-p6-porch | §6 (partial), §10 | vorch@cloud-manager | — | planned |
| P7 | ansible-p7-editor | §7 | feature@cloud-manager | — | planned |
| P8 | ansible-p8-decorations | §8 | feature@cloud-manager | P4, P5, P7 | planned |
| P9 | ansible-p9-library | §3, §9, §10 | feature@cloud-manager | P2, P5, P8 | planned |

## Execution order

P1 → P2 → P3 → P4 → P5 → P6 → P7 → P8 → P9 (serial, one plan in flight at a time; P6/P7 have
no data-series dependencies but execute in ledger order for review discipline).

## Per-plan command pair

After human review approves a plan, run (from `~/workspace/hive-deck-pro/workflow-configuration/`):

```
python3 headless-run.py approve <workflowRef> /home/todd/workspace/cloud-manager/planning/workflow-plans/cloud-manager/<codename>/PLAN.md
python3 headless-run.py exec    cloud-manager <codename>
```

Concretely, in order:

| # | approve | exec |
|---|---|---|
| P1 | `headless-run.py approve feature@cloud-manager …/cloud-manager/ansible-p1-inventory/PLAN.md` | `headless-run.py exec cloud-manager ansible-p1-inventory` |
| P2 | `headless-run.py approve feature@cloud-manager …/cloud-manager/ansible-p2-decomposition/PLAN.md` | `headless-run.py exec cloud-manager ansible-p2-decomposition` |
| P3 | `headless-run.py approve feature@cloud-manager …/cloud-manager/ansible-p3-roundtrip/PLAN.md` | `headless-run.py exec cloud-manager ansible-p3-roundtrip` |
| P4 | `headless-run.py approve feature@cloud-manager …/cloud-manager/ansible-p4-variables/PLAN.md` | `headless-run.py exec cloud-manager ansible-p4-variables` |
| P5 | `headless-run.py approve feature@cloud-manager …/cloud-manager/ansible-p5-resolution/PLAN.md` | `headless-run.py exec cloud-manager ansible-p5-resolution` |
| P6 | `headless-run.py approve vorch@cloud-manager …/cloud-manager/ansible-p6-porch/PLAN.md` | `headless-run.py exec cloud-manager ansible-p6-porch` |
| P7 | `headless-run.py approve feature@cloud-manager …/cloud-manager/ansible-p7-editor/PLAN.md` | `headless-run.py exec cloud-manager ansible-p7-editor` |
| P8 | `headless-run.py approve feature@cloud-manager …/cloud-manager/ansible-p8-decorations/PLAN.md` | `headless-run.py exec cloud-manager ansible-p8-decorations` |
| P9 | `headless-run.py approve feature@cloud-manager …/cloud-manager/ansible-p9-library/PLAN.md` | `headless-run.py exec cloud-manager ansible-p9-library` |

(`…` = `/home/todd/workspace/cloud-manager/planning/workflow-plans`. Each plan's deck.md Branch
stays TBD until its own Task 0000 mints the execution branch.)

## Epic coverage map

Every epic section → owning plan(s). "existing" = already on main (marketplace/blueprint series
and prior ansible work). Explicit deferrals listed below the map.

| Epic § | Requirement | Owner |
|---|---|---|
| §1 | Purpose & scope (umbrella) | series collectively |
| §2 | Decompose into addressable records | P2 (plays/tasks/handlers/var blocks); P1 (inventories/host+group vars) |
| §2 | Version everything | existing (`pbrev`/`rfrev`) + P2 (revision-scoped records → record diffing) |
| §2 | Explicit relationship graph | P2 |
| §2 | Reverse indexes (used-by) | P2 |
| §2 | Round-trip fidelity (ruamel) | P3 |
| §3 | Roles as library objects w/ metadata | P9 (metadata fields) + P4 (interface kinds) |
| §3 | Clean parameter interface | P4 (required/default/internal) + P9 (composer renders it) |
| §3 | Composition & dependency chains, cycle detection | P2 (edges + cycle check) + P9 (composer) |
| §3 | Reuse beyond roles | P1 (inventory groups), P2 (addressable tasks/var blocks); cross-playbook task-block library **deferred** (D1) |
| §3 | Impact-aware editing | P2 (data) + P9 (warnings UX) |
| §4 | Variable registry w/ precedence awareness | P4 (registry) + P5 (precedence engine) |
| §4 | Sensitive-parameter typing | P4 |
| §4 | No plaintext secrets (refs + Vault) | P4 (enforced); existing porch SecretInputs mechanism reused |
| §4 | Required-var enforcement | P4 + P5 (gap wiring) |
| §5 | Effective-value preview | P5 |
| §5 | Provenance | P5 |
| §5 | Operation context | P5 |
| §5 | Gap detection (ideally wired to --check) | P5; --check exists via P6 — deep gap↔check correlation **deferred** (D2) |
| §6 | Stable MCP tool surface | existing 86 tools + P1/P2/P4 (list/get), P5 (resolve), P6 (lint/check/stream) |
| §6 | Async execution (job_id pattern) | existing (runId + polling, shipped on main) |
| §6 | Streaming output | P6 (chunked log streaming) |
| §6 | Idempotent, safe tools | existing conventions, restated contracts §11, applied per plan |
| §7 | Real editor (CM6), YAML highlighting, Jinja2, fragments, inline lint, editor round-trip | P7 (lint findings sourced from P6; write fidelity from P3) |
| §8 | Semantic decorations, meaningful highlighting, hover provenance, masking everywhere | P8 |
| §9 | Library browser + graph | P9 |
| §9 | Playbook composer | P9 |
| §9 | Effective-config preview | P9 (pane) over P5 (engine) |
| §9 | Secret indicators | P8 (`<SecretValue>`, lock chips) |
| §9 | Run console (live streaming, per-task) | P9 (UI) over P6 (chunks) |
| §10 | Idempotency visibility / check-mode results | P6 (mode) + P9 (labelling) |
| §10 | Guardrails (confirm, used-by warnings, lint gates) | P9 (UX) over P2 (data) + P6 (lint) |
| §10 | Full audit trail | existing (runs/targets/events) + P5 (`rsnap` provenance depth) |
| §10 | Dry-run first | P6 |
| §11 | Security (refs only, least privilege, redaction, auditable) | P4/P5/P8 + standing porch redaction/two-token strategy |
| §11 | Performance (large inventories, deep chains) | P5 (perf-gated phase, caps) |
| §11 | Extensibility (stable schemas, clean model) | ANSIBLE-EPIC-CONTRACTS.md itself + per-plan conformance |
| §11 | Data integrity (round-trip + reverse-index invariants) | P3 (fidelity check) + P2 (transactional edges) |

### Explicit deferrals

| # | Deferred item | Rationale |
|---|---|---|
| D1 | Cross-playbook reusable task-block/var-set library (§3 full extent) | Needs P3 structured splice proven in production first; records are addressable after P2, library UX revisit post-series |
| D2 | Gap detection correlated with --check results per target (§5) | P5 gaps + P6 check both ship; the correlation layer is additive polish, post-series |
| D3 | Dedicated inventory management UI | P1 ships APIs/MCP; P9 includes inventories in search only — full pages are a follow-up plan |
| D4 | Composer "recompose existing playbook" | P9 default: new-playbook only; recompose needs P3 end-to-end maturity |
| D5 | SignalR push for log chunks | P9 polls `afterSeq`; hub push if console feel demands it |
| D6 | Execution against external (non-VM) inventory hosts | P1 stores them; porch targets VMs only — execution support is its own vorch plan |
| D7 | Lint hard-gating + `enforceRequired` on by default | Both ship opt-in (P6/P5); defaults flip after P9 gives operators visibility |
