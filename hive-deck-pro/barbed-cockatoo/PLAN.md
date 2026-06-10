# Plan: Fix the MCP plan-workflow integration — contract field mismatch and builtin plan fallback

## Objective

Make `hv_orchestrate_plan` (the MCP planning entry point) work end-to-end without per-deck ceremony. Today it always fails with "workflow_name is required" because the TypeScript tool and the Go dispatch contract disagree on a field name, and even with that fixed, plan assembly errors for any deck lacking a `decks/workflows/plan/<deck>/<name>.yaml` — even though a complete generic plan base ships builtin. End state: the MCP plan tool works for every deck out of the box, deck extensions remain optional overlays, and the MCP↔CLI contract is audited so no other tool has a silent field mismatch.

## Background

Found live on 2026-06-10 while creating a plan for the cloud-manager deck:

1. **Field mismatch (verified):** `mcp/src/tools/workflow_plan.ts` dispatches `{op: "orchestrate_plan", deck, name: workflowName}` but Go unmarshals into `WorkflowPlanInput` whose tag is `workflow_name` (`ops/contract.go:81-85`). The name silently arrives empty and `RunWorkflow` rejects at `ops/workflow.go:51-53`. Every invocation of the MCP plan tool fails.
2. **Missing builtin fallback (verified):** `internal/builtin/plan/_base.yaml` defines the complete generic plan (`@write-plan`, `@write-deck`, `@write-init`, `@ship-plans`, `@plan-overview`, `@post-ship-reminder`), but `RunWorkflow` requires a per-deck extension yaml to exist before it will assemble anything. The intent was always: builtin generic plan, per-deck extension optional. Workaround applied on 2026-06-10: an empty-extension `decks/workflows/plan/cloud-manager/plan.yaml` (`tokens: {}`, `fragments: []`) was added to workflow-configuration — it is in the working tree and ships with this plan.
3. **Default-name drift (verified):** the TS tool defaults the workflow name to `"planning"`, but the existing extension on disk is `plan.yaml` (deck hive-deck-pro). Both fixes above would still miss if the default doesn't match the convention.
4. **Stale-server confusion (observed):** the running MCP server exposed the old tool name `hv_plan` while `mcp/dist` on disk registers `hv_orchestrate_plan` — dist had been rebuilt without restarting the server. Not a code bug; the rebuild-mcp-artifacts rule already covers it. Reinforce in docs if trivial, otherwise ignore.

## Design

### Fix 1 — align the dispatch payload (mcp repo, TS side)

The Go contract (`ops/contract.go`) is canonical. Change `workflow_plan.ts` to send `workflow_name` instead of `name`. Audit every other tool in `mcp/src/tools/` against the corresponding Go input struct tags (`workflow_create`, `workflow`, `workflow_run`, `list`, `stash`, etc.) and fix any other mismatched field in the same way. Add/extend a contract test if the repo has a test harness for the dispatch layer; otherwise verify by round-trip.

### Fix 2 — builtin fallback for plan assembly (hive-deck-pro repo, Go side)

In `ops/workflow.go` `RunWorkflow`: for `type == "plan"`, when no extension file resolves (both search locations miss), assemble from the builtin base with an empty extension instead of returning an error. Scope: **plan type only** — `workflow` type extensions carry deck-specific MCPs/build/deploy fragments and must keep failing loudly when missing. The empty-extension behavior must be identical to an on-disk `tokens: {}` / `fragments: []` file (the cloud-manager workaround proves that shape works). The error message for missing *workflow*-type extensions stays as is.

### Fix 3 — align the default name

Change the TS default from `"planning"` to `"plan"` to match the on-disk convention (`plan.yaml`). With Fix 2, decks without any extension also resolve via the builtin.

### Rebuild + restart

Per hive-deck-internal-rules: rebuild the Go binary, rebuild `mcp/dist`, restart the MCP server before verifying.

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000  | Setup | hv_status + hv_init/hv_next |
| 0001  | Contract alignment + builtin plan fallback | Fix `workflow_plan.ts` field + default name; audit all MCP tool payloads against `ops/contract.go`; add builtin fallback for plan type in `ops/workflow.go`; rebuild Go + dist |
| 0002  | Verify end-to-end | `hv plan <deck> plan` works for a deck with no extension; MCP `hv_orchestrate_plan` round-trips for cloud-manager and hive-deck-pro; missing *workflow*-type extension still errors clearly |

## Open questions

- Should `hv promote --list` / plan listing also surface the builtin `plan` workflow for decks with no extension file, so discovery matches the new behavior? (Recommended: yes, label it `builtin`.)
- The planning PR auto-merged on ship for cloud-manager (PR #30). If a human-review pause is wanted for plan PRs, that is a separate behavior change — not in scope here, flagged for the owner.
