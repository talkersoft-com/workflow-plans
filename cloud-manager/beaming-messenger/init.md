# Vault-backed database connections — resolver wiring + provisioning investigations

## What this workflow does
Fixes the one verified bug from the 2026-06-10 pg-test connectivity session — the database-toolkit's Vault credential resolver exists but is imported by nothing, so vault-backed db configs (`vaultPath` entries like `pg-test`) fail with a missing-fields config error — and runs two scoped investigations (postgres VM network provisioning, double-encoded per-VM SSH secret) whose deliverables are findings documents, not speculative code changes. End state: `db_test_connection pg-test` works through an SSH tunnel, and we have evidence-based recommendations for the other two findings.

## Read before starting
- `deck.md` — which deck and repos are in scope; pre-written hv MCP calls
- `Orchestrate/ORCH.md` — full task list and execution instructions
- `../../common/toolkit/hive-deck.md` — hv MCP call patterns
- `PLAN.md` — design, acceptance criteria, and the verified-vs-investigation split

## Constraints
- Do not modify vorch-lib, vorch-service, or porch (deck convention). If an investigation points there, the deliverable is a flagged recommendation, not a change.
- Non-vault database configs (`clouddb`, `appdb`, `clouddb_local`) must behave exactly as before the resolver wiring.
- Never log secret material (tokens, passwords, private keys); a Vault failure on one config entry must not break other entries.
- Phases 0002 and 0003 produce findings documents; code changes only where the investigation confirms ownership (0002, scoped) or never (0003).
- Keep task/test count proportional — this is one small code change plus two investigations.
