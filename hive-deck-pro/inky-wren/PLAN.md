# Plan: root_mcp_mode flag — overwrite vs merge for workspace root .mcp.json

## Objective

Add a `root_mcp_mode: overwrite | merge` flag to `mcp_manager` in `config.yaml`. Default is `overwrite` — the workspace root `.mcp.json` is replaced fresh on every `hv init` / `hv mcp` run, eliminating stale entries. `merge` preserves the existing behaviour for users who deliberately run multiple decks simultaneously.

## Background

The workspace root `.mcp.json` is written by `internal/mcp/mcp.go:Apply()` using `writeFile(rootFile, servers, merge=true)`. The merge mode accumulates servers from all decks but never removes or updates existing entries. This caused the `HV_HOME` drift we hit — an outdated value persisted in the file because merge never overwrote it. Since most users run one deck at a time, `overwrite` is the safer default.

## Design

**`internal/config/config.go`** — extend `MCPManagerConfig`:
```go
type MCPManagerConfig struct {
    Enabled     bool   `yaml:"enabled"`
    RootMCPMode string `yaml:"root_mcp_mode"` // "overwrite" | "merge"; default "overwrite"
}
```

**`internal/mcp/mcp.go`** — `Apply` signature gains the mode:
```go
func Apply(decksRoot string, l *config.Loaded) error {
    ...
    merge := l.Setup.MCPManager.RootMCPMode == "merge"
    writeFile(rootFile, servers, merge)
}
```

When `RootMCPMode` is empty, default to `"overwrite"`.

**`config.yaml.example`** — add the new field with a comment:
```yaml
mcp_manager:
  enabled: true
  root_mcp_mode: overwrite   # overwrite | merge — overwrite replaces root .mcp.json fresh each run; merge accumulates across decks
```

No other files change — `Apply` already threads through `config.Loaded`.

## Implementation phases

| Phase | Title      | Description                                                                 |
|-------|------------|-----------------------------------------------------------------------------|
| 0000  | Setup      | hv_status + hv_next                                                         |
| 0001  | Implement  | Add `RootMCPMode` to config, update `Apply()` to use it, update example    |
| 0002  | Build+Test | `make install`; run `hv mcp hive-deck-pro` and confirm root `.mcp.json` is replaced cleanly |

## Open questions

None.
