# Plan: hive-deck JSON CLI Refactor — Operations Contract

## Objective

Refactor hive-deck-pro so the Go core becomes a JSON CLI library: every operation accepts a structured JSON payload as input and returns the existing text output unchanged. The cobra CLI becomes a thin arg-to-JSON wrapper; the MCP bypasses cobra and calls the JSON CLI directly. An operations contract — JSON Schema + markdown — is committed to the repo as the authoritative input specification. The end state is functionally 1:1 with the current version; only the internal wiring changes.

## Background

Currently the MCP (`mcp/src/runner.ts`) invokes `hv <subcommand> [args...]` and constructs positional argument arrays. This is brittle — adding a new flag requires changes in three places (cobra command, MCP runner, MCP tool). The cobra CLI and MCP are tightly coupled through CLI argument conventions with no formal contract.

The JSON CLI pattern decouples these layers:
- The core Go logic exposes a single dispatch point: read a JSON payload, route to the correct operation handler, return text output
- The cobra CLI becomes a pure translation layer: parse human-typed args → JSON → dispatch
- The MCP constructs JSON directly → dispatch, skipping cobra entirely
- The operations contract markdown file makes the JSON shape explicit and versionable

This also unlocks future extensibility: any new caller (CI scripts, other MCPs, web hooks) can drive hive-deck by sending JSON without reimplementing CLI argument parsing.

## Design

### Folder structure

```
hive-deck-pro/
├── cmd/hv/main.go              ← entry point unchanged (cobra wrapper)
├── ops/                        ← NEW: JSON CLI backend
│   ├── dispatch.go             ← ReadJSON → route → handler → output
│   ├── contract.go             ← typed input structs for every operation
│   ├── init.go                 ← InitOp handler
│   ├── status.go               ← StatusOp handler
│   ├── ship.go                 ← ShipOp handler
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
│   ├── schema.json             ← JSON Schema for all operations
│   └── CONTRACT.md             ← human-readable operations contract
├── internal/                   ← existing packages unchanged
│   ├── config/
│   ├── resolve/
│   ├── provision/
│   └── ...
└── mcp/src/runner.ts           ← updated: calls hv --json instead of hv <args>
```

### Operations contract shape

Each operation has a named input type. The JSON payload always has an `op` discriminator field:

```json
{ "op": "status", "deck": "cloud-manager" }
{ "op": "init",   "deck": "cloud-manager" }
{ "op": "ship",   "deck": "cloud-manager", "message": "fix: ...", "title": "fix: ..." }
{ "op": "next",   "deck": "cloud-manager" }
{ "op": "sync",   "deck": "cloud-manager" }
{ "op": "prune",  "deck": "cloud-manager" }
{ "op": "stash",  "deck": "cloud-manager" }
{ "op": "unstash","deck": "cloud-manager" }
{ "op": "teardown","deck": "cloud-manager" }
{ "op": "workflow","deck": "cloud-manager", "name": "create-workflow", "list": false }
{ "op": "decks" }
{ "op": "list",   "deck": "cloud-manager" }
{ "op": "mcp",    "deck": "cloud-manager" }
{ "op": "branch", "deck": "cloud-manager" }
```

### JSON CLI invocation

```bash
# Via stdin (MCP path — preferred, no shell quoting issues)
echo '{"op":"status","deck":"cloud-manager"}' | hv --json

# Via --json flag (scripting path)
hv --json '{"op":"status","deck":"cloud-manager"}'
```

### `ops/dispatch.go`

```go
func Dispatch(payload []byte) (string, error) {
    var base struct { Op string `json:"op"` }
    if err := json.Unmarshal(payload, &base); err != nil {
        return "", fmt.Errorf("invalid JSON: %w", err)
    }
    switch base.Op {
    case "status":  return runStatus(payload)
    case "init":    return runInit(payload)
    case "ship":    return runShip(payload)
    // ...etc
    default:
        return "", fmt.Errorf("unknown op %q", base.Op)
    }
}
```

### Cobra wrapper pattern

Each cobra command becomes a marshal-and-dispatch call:

```go
// Before:
runE: func(cmd *cobra.Command, args []string) error {
    deck := args[0]
    return doStatus(deck)
}

// After:
runE: func(cmd *cobra.Command, args []string) error {
    payload, _ := json.Marshal(ops.StatusInput{Op: "status", Deck: args[0]})
    out, err := ops.Dispatch(payload)
    if err != nil { return err }
    fmt.Print(out)
    return nil
}
```

### MCP runner update

```typescript
// Before:
export function runHv(...args: string[]): RunResult {
  const result = spawnSync(HV_BIN, args, { ... });
  ...
}

// After:
export function runHv(payload: object): RunResult {
  const json = JSON.stringify(payload);
  const result = spawnSync(HV_BIN, ["--json"], {
    input: json,
    encoding: "utf-8",
    stdio: ["pipe", "pipe", "pipe"],
  });
  ...
}
```

Each MCP tool constructs a payload object instead of an arg array:

```typescript
// Before: runHv("status", deck)
// After:  runHv({ op: "status", deck })
```

### CONTRACT.md

Committed at `ops/contract/CONTRACT.md`. Documents every operation: op name, required fields, optional fields, example payload, example output. Not gitignored — this is a first-class project artifact.

### schema.json

`ops/contract/schema.json` — JSON Schema (draft-07) with a `oneOf` discriminated on `"op"`. Each branch validates the fields for that operation. The `dispatch.go` does NOT validate against the schema at runtime (too slow); the schema is for tooling and documentation only.

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000 | Setup | hv_status + hv_next |
| 0001 | ops/ package + contract | Create `ops/` package with `contract.go` (all input structs) and `dispatch.go` (routing skeleton). Write `CONTRACT.md` and `schema.json`. No handlers wired yet — dispatch returns "not implemented" for all ops. |
| 0002 | Port handlers | Move logic from `cmd/hv/main.go` cobra RunE bodies into `ops/*.go` handlers. Wire each into dispatch. Build + all existing `go test ./...` pass. |
| 0003 | Slim cobra wrapper | Replace cobra RunE bodies with marshal→dispatch→print. `cmd/hv/main.go` becomes a pure translation layer. Add `--json` flag to root command. `hv --json '...'` and `echo '...' \| hv --json` both work. |
| 0004 | Update MCP runner | Change `runHv(...args)` to `runHv(payload: object)` in `runner.ts`. Update every call site in MCP tools. Rebuild MCP. End-to-end test: `hv_status`, `hv_ship`, `hv_workflow` all work via JSON path. |
| 0005 | Ship | Tests pass, contract files committed, make install works, ship hive-deck-pro. |

## Open questions

- **`--json` flag vs subcommand**: Using `hv --json <payload>` keeps the binary name and makes the JSON path opt-in. Alternative is a dedicated `hv json` subcommand. Flag is cleaner since it avoids polluting the subcommand list. Recommend flag.
- **Stdin vs flag for JSON input**: Supporting both (stdin when `--json` has no value, flag value when provided) gives the most flexibility. MCP uses stdin to avoid shell quoting; humans use the flag. Recommend both.
- **Schema validation at runtime**: Validating JSON Schema on every dispatch adds ~5ms and a dependency. Not worth it for v1 — the typed Go structs provide sufficient validation. Schema is docs-only.
- **gitignore of CONTRACT.md**: Contract should NOT be gitignored — it is a first-class project artifact. The existing gitignore rule `*.md` in the deck config applies to the workspace folder, not the repo contents. No action needed.
