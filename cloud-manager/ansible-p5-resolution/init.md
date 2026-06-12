# Resolution engine — Epic plan 5 (ansible-p5-resolution)

## What this workflow does
Builds the precedence-aware resolution engine: effective value + provenance (winner and
overridden chain) for every variable in a concrete playbook+inventory+host context, secret
masking at the engine level, required-var gap detection wired to the run trigger, a per-run
ResolutionSnapshot audit record, and the `cloud_resolution_preview` MCP tool. Exposes C-RES,
which P8 hover-provenance and P9 preview panes consume.

## Read before starting
- `deck.md`, `PLAN.md` — engine design, masking stance, perf caps
- `../../master-plans/ANSIBLE-EPIC-CONTRACTS.md` — §7 C-VAR (consumed verbatim), §8 C-RES
  (exposed verbatim — binding)
- `../ansible-p1-inventory/PLAN.md`, `../ansible-p2-decomposition/PLAN.md`,
  `../ansible-p4-variables/PLAN.md` — the record layers the engine folds
- `../../master-plans/ANSIBLE-EPIC.md` — epic §5, §6, §10, §11

## Constraints
- **Series order**: requires P1, P2, P4 merged.
- **C-RES is binding verbatim** (contracts §8): field names, `"***"` masking literal, and the
  never-return-secret-cleartext invariant live in the ENGINE, not in controllers.
- Resolution is always operation-scoped (playbook+inventory+host) — no abstract global resolve.
- No vorch-lib/vorch-service/porch changes; no web changes.
- Performance is a requirement, not a nicety (epic §11): the synthetic-scale perf test gates the
  final phase.
- Existing run flows byte-identical when the flag is off — regression-check.
