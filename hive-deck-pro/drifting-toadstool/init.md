# Shared config layer + canonical path tokens (drifting-toadstool)

## What this workflow does
Implements R7.3–R7.6: machine `config.yaml` declares a `deck_config:` shared layer; every config
artifact (deck yamls, workflow extensions, fragments, modules, profiles, rulesets) resolves
machine-first then layer, per file, with union scans for listings and the workflow registry;
`config.yaml` itself never falls back. Adds the canonical `{{WorkspaceRoot}}` token (legacy
`{{DecksRoot}}` alias kept), `~` expansion, and relative-to-declaring-file paths across config
and mcps.yaml registries.

## Read before starting
- `deck.md` — scope and pre-written hv MCP calls
- `Execution/Exec.md` — full task list and execution instructions
- `PLAN.md` — before/after contracts, resolution rules, open-question defaults

## Constraints
- **Back-compat is the acceptance bar**: a config.yaml without `deck_config` must behave
  byte-identically to today — including assembled plan/workflow output (golden comparison in the
  verification phase). `{{DecksRoot}}` keeps working everywhere it does now.
- Per-FILE fallback only — never key-level merging of yaml files across layers.
- `config.yaml` is machine identity: machine root only, no layer fallback, ever.
- hive-deck internal rules apply: rebuild the Go binary AND mcp/dist, restart the MCP server
  before verification; tests proportional to each phase.
- workflow-configuration restructuring is OUT of scope (follow-up config change).
