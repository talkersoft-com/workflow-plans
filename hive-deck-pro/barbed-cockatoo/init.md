# Fix MCP plan-workflow integration — contract mismatch + builtin plan fallback

## What this workflow does
Repairs the hive-deck MCP planning path, which currently always fails: the TS tool sends `name` where the Go dispatch contract expects `workflow_name`, and plan assembly demands a per-deck extension yaml even though the generic plan base is builtin. After this workflow, `hv_orchestrate_plan` works for any deck with zero per-deck setup, deck plan extensions remain optional overlays, and all MCP tool payloads are verified against `ops/contract.go`.

## Read before starting
- `deck.md` — which deck and repos are in scope; pre-written hv MCP calls
- `Orchestrate/ORCH.md` — full task list and execution instructions
- `../../common/toolkit/hive-deck.md` — hv MCP call patterns
- `PLAN.md` — the three verified bugs, design, and scope limits

## Constraints
- The Go contract (`ops/contract.go`) is canonical — fix the TypeScript side to match it, not the reverse.
- Builtin fallback applies to plan-type workflows only; missing workflow-type extensions must keep erroring loudly (they carry deck-specific MCP/build/deploy fragments).
- After any change: rebuild the Go binary AND `mcp/dist`, then restart the MCP server before verification (hive-deck-internal-rules).
- Keep task/test count proportional — two small code fixes plus an audit and verification; do not expand scope into the plan-PR approval behavior (flagged as an open question only).
