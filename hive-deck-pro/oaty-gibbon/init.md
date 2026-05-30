# hive-deck-feature

## What this workflow does
Implements the per-deck `branch:` field across the hive-deck-pro Go codebase and all deck YAML files. Removes `default_target_branch` and `target_branches` from config.yaml and the Go Setup struct, adds `DeckFile.Branch`, updates resolve/provision/branch/ship to use it, and writes `branch: hive` into every deck YAML that targets the hive branch.

## Read before starting
- `deck.md` — which deck and repos are in scope; pre-written hv MCP calls
- `PLAN.md` — full design, before/after examples, phase breakdown
- `../../common/toolkit/hive-deck.md` — hv MCP call patterns

## Constraints
- Do not introduce per-repo branch overrides of any kind — one branch per deck is the invariant
- `go build ./...` must pass with zero errors after every phase
- `make install` must succeed before shipping
