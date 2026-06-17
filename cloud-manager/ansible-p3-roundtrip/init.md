# Round-trip YAML service — Epic plan 3 (ansible-p3-roundtrip)

## What this workflow does
Adds a ruamel.yaml round-trip engine as a Python sidecar beside the API (decision recorded in
PLAN.md: sidecar, NOT porch helper), a `ydoc` fidelity record per revision with a write-time
`recompose(decompose(x)) == x` check, and the structured-edit write path (PATCH play/task) that
splices changes into the user's YAML without touching formatting, comments, or key order.

## Read before starting
- `deck.md`, `PLAN.md` — especially the sidecar-vs-porch decision table and rationale
- `../ansible-p2-decomposition/PLAN.md` — the read-model stance this plan complements
- `../../master-plans/ANSIBLE-EPIC-CONTRACTS.md` — §1 (`ydoc`), §3 round-trip routes
- `../../master-plans/ANSIBLE-EPIC.md` — epic §2 (round-trip fidelity), §11 (data integrity)

## Constraints
- **Series order**: requires P2 merged (records + decomposer exist to integrate with).
- **No porch/vorch changes** — the sidecar decision makes this a hard constraint, not a
  preference. No new queues or messages.
- Fidelity is an invariant: byte-identical round-trip enforced at write time; golden-file corpus
  must pass before the structured-edit endpoints ship.
- Structured edits go through the EXISTING revision-cutting path so history/events/decomposition
  fire unchanged.
- Code changes in cloud-manager-api repo only (including the new `roundtrip-service/` folder).
