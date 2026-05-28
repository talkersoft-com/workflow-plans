# Plan: hive-deck JSON CLI Refactor — Operations Contract

## Objective

Extract all hive-deck operation logic into an `ops/` Go library package. The cobra CLI continues to call `ops/` directly via Go imports — no change to how humans use `hv`. The `ops/` package additionally exposes a JSON dispatch entry point (`ops.Dispatch([]byte) string`) so non-Go callers can drive any operation by sending a JSON payload to stdin. The MCP (`mcp/src/runner.ts`) is updated to use this JSON stdin path instead of building cobra arg arrays. An operations contract (`ops/contract/CONTRACT.md` + `schema.json`) is committed to the repo documenting every operation's JSON input shape. The result is 1:1 functionally with the current version.

## Background

Currently all operation logic lives directly in `cmd/hv/main.go` cobra `RunE` bodies. The MCP calls `hv <subcommand> [args...]`, constructing positional arg arrays. This creates two problems:

1. **No formal contract**: there is no machine-readable spec of what inputs each operation accepts. Adding a flag requires touching cobra, the MCP runner, and the MCP tool separately.
2. **Language boundary friction**: the MCP (TypeScript) has no way to call Go functions directly — it must go through the CLI. But the CLI was designed for human use, not programmatic use. Arg arrays are fragile; JSON is structured.

The fix: extract operation logic into `ops/`, keep cobra as a direct Go caller, expose a JSON dispatch path for the MCP. The JSON CLI interface exists **only because the MCP cannot import Go** — it is not a second user-facing CLI style. Cobra stays the human interface.

## Design

### Folder structure

```
hive-deck-pro/
├── cmd/hv/main.go              ← cobra CLI — unchanged user interface
│                                  RunE bodies call ops.* directly via Go import
├── ops/                        ← NEW: core Go library
│   ├── dispatch.go             ← JSON entry point: ops.Dispatch([]byte) (string, error)
│   ├── contract.go             ← typed input structs for every operation (shared by dispatch + cobra)
│   ├── init.go                 ← RunInit(input InitInput) (string, error)
│   ├── status.go               ← RunStatus(input StatusInput) (string, error)
│   ├── ship.go                 ← RunShip(input ShipInput) (string, error)
│   ├── sync.go
│   ├── prune.go
│   ├── next.go
│   ├── stash.go
│   ├── unstash.go
│   ├── teardown.go
│   ├── workflow.go
│   ├── decks.go
│   ├── list.go
│   ├── mcp.go
│   └── branch.go
├── ops/contract/
│   ├── schema.json             ← JSON Schema (draft-07) — one entry per op
│   └── CONTRACT.md             ← human-readable operations contract (NOT gitignored)
├── internal/                   ← existing packages — unchanged
│   ├── config/
│   ├── resolve/
│   └── ...
└── mcp/src/runner.ts           ← updated: sends JSON to hv stdin, reads stdout
```

### Two caller paths — same ops/ library

```
Human / terminal:
  hv status cloud-manager
    → cmd/hv cobra RunE
    → ops.RunStatus(StatusInput{Deck: "cloud-manager"})   ← direct Go call
    → string output printed

MCP / TypeScript:
  runHv({ op: "status", deck: "cloud-manager" })
    → JSON marshaled to stdin
    → hv binary reads stdin, calls ops.Dispatch(payload)
    → ops.Dispatch routes to ops.RunStatus(...)
    → string output returned to MCP
```

The JSON path is an **additional entry point on the same binary** — not a separate binary or mode flag. The binary detects stdin is piped JSON and dispatches accordingly (or: a minimal `--json` flag triggers the dispatch path — implementation detail resolved in Phase 0001).

### ops/contract.go — input structs

```go
package ops

type StatusInput  struct { Op string `json:"op"` ; Deck string `json:"deck"` }
type InitInput    struct { Op string `json:"op"` ; Deck string `json:"deck"` }
type ShipInput    struct { Op string `json:"op"` ; Deck string `json:"deck"` ; Message string `json:"message"` ; Title string `json:"title"` ; Body string `json:"body,omitempty"` }
type NextInput    struct { Op string `json:"op"` ; Deck string `json:"deck"` }
type SyncInput    struct { Op string `json:"op"` ; Deck string `json:"deck"` }
type PruneInput   struct { Op string `json:"op"` ; Deck string `json:"deck"` }
type StashInput   struct { Op string `json:"op"` ; Deck string `json:"deck"` }
type UnstashInput struct { Op string `json:"op"` ; Deck string `json:"deck"` }
type TeardownInput struct{ Op string `json:"op"` ; Deck string `json:"deck"` }
type WorkflowInput struct{ Op string `json:"op"` ; Deck string `json:"deck"` ; Name string `json:"name,omitempty"` ; List bool `json:"list,omitempty"` }
type DecksInput   struct { Op string `json:"op"` }
type ListInput    struct { Op string `json:"op"` ; Deck string `json:"deck"` }
type McpInput     struct { Op string `json:"op"` ; Deck string `json:"deck"` }
type BranchInput  struct { Op string `json:"op"` ; Deck string `json:"deck"` }
```

### ops/dispatch.go — JSON entry point

```go
func Dispatch(payload []byte) (string, error) {
    var base struct{ Op string `json:"op"` }
    if err := json.Unmarshal(payload, &base); err != nil {
        return "", fmt.Errorf("invalid JSON payload: %w", err)
    }
    switch base.Op {
    case "status":
        var in StatusInput
        json.Unmarshal(payload, &in)
        return RunStatus(in)
    case "init":
        var in InitInput
        json.Unmarshal(payload, &in)
        return RunInit(in)
    case "ship":
        var in ShipInput
        json.Unmarshal(payload, &in)
        return RunShip(in)
    // ... all ops
    default:
        return "", fmt.Errorf("unknown op %q — see ops/contract/CONTRACT.md", base.Op)
    }
}
```

### cmd/hv cobra — direct Go call (unchanged interface)

```go
// status cobra command — before:
RunE: func(cmd *cobra.Command, args []string) error {
    // inline logic here
}

// status cobra command — after:
RunE: func(cmd *cobra.Command, args []string) error {
    out, err := ops.RunStatus(ops.StatusInput{Deck: args[0]})
    if err != nil { return err }
    fmt.Print(out)
    return nil
}
```

User experience: `hv status cloud-manager` works identically.

### JSON stdin entry point in cmd/hv/main.go

```go
func main() {
    // If stdin is piped, treat it as a JSON dispatch call
    stat, _ := os.Stdin.Stat()
    if (stat.Mode() & os.ModeCharDevice) == 0 {
        payload, _ := io.ReadAll(os.Stdin)
        out, err := ops.Dispatch(payload)
        if err != nil { fmt.Fprintln(os.Stderr, "error:", err); os.Exit(1) }
        fmt.Print(out)
        return
    }
    // Otherwise: cobra CLI
    rootCmd.Execute()
}
```

No flag needed — piped stdin triggers JSON mode automatically. Human use (`hv status cloud-manager`) never pipes stdin so cobra always wins.

### MCP runner update

```typescript
// runner.ts — before:
export function runHv(...args: string[]): RunResult {
  const result = spawnSync(HV_BIN, args, { encoding: "utf-8", stdio: ["ignore","pipe","pipe"] });
  ...
}

// runner.ts — after:
export function runHv(payload: object): RunResult {
  const result = spawnSync(HV_BIN, [], {
    input: JSON.stringify(payload),
    encoding: "utf-8",
    stdio: ["pipe", "pipe", "pipe"],
  });
  ...
}
```

Each MCP tool replaces arg arrays with payload objects:
```typescript
// Before: runHv("status", deck)
// After:  runHv({ op: "status", deck })

// Before: runHv("ship", deck, "--message", message, "--title", title)
// After:  runHv({ op: "ship", deck, message, title })
```

### ops/contract/CONTRACT.md

Committed, not gitignored. Documents every op: name, required fields, optional fields, example JSON payload, example output. This is the authoritative spec for the JSON interface.

### ops/contract/schema.json

JSON Schema draft-07. `oneOf` discriminated union on `"op"`. Used by tooling and documentation; not validated at runtime (typed Go structs handle that).

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000 | Setup | hv_status + hv_next |
| 0001 | ops/ skeleton + contract | Create `ops/` package with `contract.go` (all input structs) and `dispatch.go` (routing). Write `CONTRACT.md` and `schema.json`. Handlers return `"not implemented"` stubs. `go build` passes. |
| 0002 | Port handlers | Move logic from `cmd/hv/main.go` RunE bodies into `ops/*.go` handler functions. Wire each into dispatch. `go test ./...` passes. Cobra still calls inline logic (not yet wired to ops). |
| 0003 | Wire cobra to ops | Replace cobra RunE inline logic with `ops.Run*()` direct calls. Add stdin-piped JSON dispatch to `main()`. `hv <subcommand>` output unchanged. `echo '{"op":"status","deck":"X"}' \| hv` works. |
| 0004 | Update MCP runner | Change `runHv(...args)` → `runHv(payload: object)` in `runner.ts`. Update every MCP tool call site. Rebuild MCP. Test `hv_status`, `hv_ship`, `hv_workflow` end-to-end. |
| 0005 | Ship | All tests pass, contract files committed, `make install` works, ship hive-deck-pro. |
