# Library UX — Epic plan 9 (ansible-p9-library)

## What this workflow does
Ships the epic's user-facing payoff: a library browser with search/tags and a dependency-graph
view over P2's edge graph, a role-interface composer that renders required/default/internal
params as validated forms (secrets ref-only), per-target effective-config preview with
provenance and gaps, impact guardrails ("used by N" warnings, breaking-change checks, lint-gate
toggle), and a run console tail-following P6's log chunks. Internal split declared: Part A
(metadata + browser/graph), Part B (composer + preview + guardrails + console).

## Read before starting
- `deck.md`, `PLAN.md` — the split, the guardrail flows, the additive API exception
- `../../master-plans/ANSIBLE-EPIC-CONTRACTS.md` — §5 C-GRAPH, §7 C-VAR, §8 C-RES; P9 consumes,
  never reshapes
- `../ansible-p2-decomposition/PLAN.md` (used-by APIs), `../ansible-p5-resolution/PLAN.md`
  (preview), `../ansible-p8-decorations/PLAN.md` (C-DECO shared components this plan reuses)
- `../../master-plans/ANSIBLE-EPIC.md` — epic §3, §9, §10

## Constraints
- **Series order**: requires P2, P5, P8 merged (terminal plan; runs last in ledger order).
- Guardrails MUST go through P2's used-by APIs — no client-side reference guessing.
- The only non-web diff is the additive `ansible_roles` metadata change (no new entities, no
  prefix registry changes); everything else is cloud-manager-web.
- Secrets stay refs + masked end-to-end: composer writes `$secretRef` forms only; preview
  renders via C-DECO's `<SecretValue>`.
- Existing `/ansible/*` and blueprint/marketplace pages byte-identical — regression-check.
- Reuse house components (Modal/ConfirmModal, StatusBadge, rjsf, SCSS tokens); no component
  library.
