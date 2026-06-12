# Playbook decomposition + entity graph — Epic plan 2 (ansible-p2-decomposition)

## What this workflow does
Derives addressable records (plays, tasks, handlers, var blocks) from stored playbook/role YAML
on every save, and materializes a typed edge graph (`depends_on`, `includes`, `consumed_by`,
`targets`, `defines`, `overrides`) with used-by reverse indexes, cycle detection, and graph
APIs. The YAML blob stays the source of truth — these records are a rebuilt-per-revision read
model; the write path (round-trip editing) is plan 3.

## Read before starting
- `deck.md`, `PLAN.md` (design stance: derived read model — read it before coding)
- `../ansible-p1-inventory/PLAN.md` — inventory records this plan's `targets` edges reference
- `../../master-plans/ANSIBLE-EPIC-CONTRACTS.md` — §1 entities, §5 edge contract
- `../../master-plans/ANSIBLE-EPIC.md` — epic §2, §3

## Constraints
- **Series order**: requires P1 merged (edges reference `inv`/`invg`/`invh` records).
- Records are NEVER recomposed to YAML in this plan — no structured-edit PATCH endpoints
  (explicitly deferred to after P3).
- Decomposition failure must not block saves; mark the revision and surface it.
- Cycle detection on `depends_on` is mandatory (epic §3); reverse-index consistency is an
  invariant — edges rebuild in the same transaction as records.
- Code changes in cloud-manager-api and cloud-manager-mcp ONLY.
- Existing playbook/role/run flows byte-identical — regression-check in the final phase.
