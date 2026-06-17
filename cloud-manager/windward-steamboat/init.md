# Ansible Epic planning factory (windward-steamboat)

## What this workflow does
Authors the nine-plan Ansible Epic series: shared contracts file first, nine dependency-ordered
plan documents (ansible-p1-inventory … ansible-p9-library), a series ledger, and a
cross-consistency pass — then ships everything as ONE plan-stage PR for batched human review and
STOPS. No code changes anywhere.

## Read before starting
- `deck.md`, `Execution/Exec.md`
- The epic: `/home/todd/workspace/hive-deck-pro/planning/workflow-plans/master-plans/ANSIBLE-EPIC.md`
  (hive-deck-pro deck is provisioned on this host)
- `../../../workflow-plans/cloud-manager/windward-steamboat/PLAN.md`
- Current-state research is mandatory before writing: entities (CloudManager.Entities), MCP
  surface, web pages, porch capabilities — plans must build on what exists, not contradict it.

## Constraints
- DOCUMENTS ONLY: the only repos that change are workflow-plans (+ this exec folder runtime).
- Contracts file BEFORE any plan; every plan cites contracts consumed/exposed; consistency pass
  is mandatory and fixes drift in place.
- Each plan: standard shape, before/after contract examples, proportional phases, own open
  questions; executable later under its stated workflow ref (P6 under vorch@cloud-manager).
- Ship stage "plan" (await_merge) and STOP after announcing the PR — the human reviews and
  approves; never proceed into any of the nine plans.
