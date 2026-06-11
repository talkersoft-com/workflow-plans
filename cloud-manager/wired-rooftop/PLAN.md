# Plan: Blueprint composer navigation — edit affordance + create-to-composer (wired-rooftop)

## Objective

Make the blueprint composer reachable the way users expect: every row on the `/blueprints` list
gets an explicit Edit affordance that opens `/blueprints/{id}`, and a successful create navigates
straight into the new blueprint's composer instead of dropping back on the list. No other builder
behavior changes.

## Background

copper-kiosk shipped the builder split as create-modal (metadata) + composer page (playbooks,
vars, lifecycle) — functionally complete and tested, but first real use found the seam: the list
renders no link to the composer (URL memory required), and create leaves you on the list with no
hint that composition lives one level deeper. The system's own designer didn't find it — end
users won't either.

## Design

- `/blueprints` list rows (BlueprintsPage, cloud-manager-web): add an Edit action per row
  navigating to `/blueprints/{id}`, following the deck's existing list→detail pattern (as the
  VMs and Playbooks lists do — clickable name plus explicit action button; match whichever both
  of those use). Existing row actions (publish/archive/delete badges and buttons) unchanged.
- Create modal success handler: `navigate(`/blueprints/{id}`)` with the public id from the create
  response, replacing the current stay-on-list behavior. Failure path unchanged (errors stay in
  the modal).
- Feature-flag gating, routes, slices, api client: all untouched. cloud-manager-web only.

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000 | Setup | hv_status + hv_init/hv_next |
| 0001 | Navigation affordances | List-row Edit → composer; create → navigate to composer; build green |
| 0002 | Verify + deploy + ship | Playwright on the deployed app (Edit lands on composer; create lands on composer; list actions regression); deploy web; RESULT/LESSONS; hv_ship |

## Open questions

None — patterns and scope are fully constrained by the existing pages.
