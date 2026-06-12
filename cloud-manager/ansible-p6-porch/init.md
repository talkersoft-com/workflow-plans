# Porch execution upgrades — Epic plan 6 (ansible-p6-porch)

## What this workflow does
Adds `--check` dry-runs (run `mode` field, no message changes), an ansible-lint pipeline (new
`playbook-lints` queue + `plint` findings per revision), and chunked log streaming (`logc`
sequenced chunks beside the existing 16KB-tail PATCH) to the porch execution path, plus
ansible-lint health probing into the `ansible-studio` flag and three MCP tools. Runs under
`vorch@cloud-manager` — the vorch workflow rules govern every change.

## Read before starting
- `deck.md`, `PLAN.md` — especially the additive-only audit table
- The vorch workflow rules for `vorch@cloud-manager` (message contracts, consumer patterns)
- `../../master-plans/ANSIBLE-EPIC-CONTRACTS.md` — §1 (`plint`/`logc`), §3, §6, §11
  (porch additive-only is a series-wide invariant)
- vorch-service `internal/ansible/` + vorch-lib `models/messages/` — current pipeline and structs
- `../../master-plans/ANSIBLE-EPIC.md` — epic §6, §10

## Constraints
- **Message contracts are ADDITIVE ONLY**: existing structs, JSON tags, queue names, and the
  five status strings are byte-identical after this plan. New capability = new struct + new
  queue, mirroring the collection-install pattern.
- The existing 16KB-tail PATCH behavior is preserved (parity-tested) — chunks are additive.
- Crash-recovery semantics (terminal-status check on redelivery, manual ack) apply to the new
  lint consumer.
- Redaction (`redact.ScrubString`) applies to chunks exactly as to the tail PATCH.
- No web changes; series-independent (may execute without P1–P5, after them in ledger order).
