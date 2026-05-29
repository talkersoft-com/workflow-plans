# Plan: remove dead deck-level mcp.json + claude_settings mode flag

## Objective

Two consistency improvements: (1) remove the dead deck-level `.mcp.json` write from `mcp.go` — Claude Code only reads from the workspace root, so this code is never useful; (2) replace `claude_settings.force_overwrite: bool` with `claude_settings.mode: overwrite | skip | merge`, consistent with `root_mcp_mode`, adding a true merge option for `.claude/settings.local.json`.

## Background

**Dead deck-level .mcp.json**: `Apply()` writes two files — a deck-specific one at `{decksRoot}/{deckName}/.mcp.json` and the workspace root. The deck-specific file is gitignored and Claude Code ignores it. It's wasted I/O and confusing output.

**force_overwrite inconsistency**: `claude_settings.force_overwrite: bool` predates `root_mcp_mode: string`. It only has two states (overwrite/skip) with no merge option. Merging claude settings across re-provisions is useful — e.g. a repo might have local permissions additions that shouldn't be stomped. Renaming to `mode` with three values makes the API consistent and extensible.

## Design

### Phase 1 — Remove deck-level .mcp.json

In `internal/mcp/mcp.go` `Apply()`, delete:
```go
deckFile := filepath.Join(decksRoot, l.DeckName, mcpFile)
if err := writeFile(deckFile, servers, false); err != nil { ... }
fmt.Printf("mcp: wrote %d server(s) to %s\n", ...)
```
Only the workspace root write remains.

### Phase 2 — claude_settings.mode

**`internal/config/config.go`** — update `ClaudeSettings`:
```go
type ClaudeSettings struct {
    Enabled         bool              `yaml:"enabled"`
    Mode            string            `yaml:"mode"`           // "overwrite" | "skip" | "merge"
    ForceOverwrite  bool              `yaml:"force_overwrite"` // deprecated
    Permissions     ClaudePermissions `yaml:"permissions"`
    DefaultMode     string            `yaml:"defaultMode"`
}
```
In `loadSetup` backwards compat shim:
```go
if cs.Mode == "" {
    if cs.ForceOverwrite {
        fmt.Fprintln(os.Stderr, "warning: claude_settings.force_overwrite is deprecated — use mode: overwrite")
        cs.Mode = "overwrite"
    } else {
        cs.Mode = "skip"
    }
}
```

**`internal/claude/claude.go`** — update `MaybeWrite()` to branch on `Mode`:
- `overwrite` — always write (current `force_overwrite: true` behaviour)
- `skip` — skip if file exists (current `force_overwrite: false` behaviour)
- `merge` — if file exists, read it, deep-merge `permissions.allow` (union of both arrays) and keep existing `defaultMode` unless the config has a non-empty one

**`config.yaml.example`** — replace `force_overwrite: true` with `mode: overwrite`.

## Implementation phases

| Phase | Title         | Description                                                                 |
|-------|---------------|-----------------------------------------------------------------------------|
| 0000  | Setup         | hv_status + hv_next                                                         |
| 0001  | Dead code     | Remove deck-level .mcp.json write from mcp.go                              |
| 0002  | Mode flag     | Add Mode to ClaudeSettings, backwards compat shim, update MaybeWrite()     |
| 0003  | Build + test  | `make install`, test all three modes, ship                                  |

## Open questions

None.
