# Blueprint composer navigation (wired-rooftop)

## What this workflow does
Two navigation affordances in cloud-manager-web: an Edit action on every /blueprints list row
opening the composer, and create-modal success navigating directly into the new blueprint's
composer. Nothing else changes.

## Read before starting
- `deck.md` — scope and hv MCP calls
- `Execution/Exec.md` — task list
- `../../../workflow-plans/cloud-manager/wired-rooftop/PLAN.md`

## Constraints
- cloud-manager-web ONLY; follow the existing VMs/Playbooks list→detail pattern; no new
  dependencies; flag gating untouched; scope is strictly the two affordances.
- Build per phase (--target web); deploy web before ship; playwright verification on the
  deployed app; proportional tests.
