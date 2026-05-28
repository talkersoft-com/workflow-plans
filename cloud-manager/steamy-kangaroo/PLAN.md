# Plan: hv_ship — await_merge infinite loop + open_browser flag

## Objective

Simplify `await_merge` back to a single-call infinite polling loop (no `continue_transaction` split), and add an `open_browser` config flag that opens a browser tab for each PR URL immediately after they are created — giving the user a direct, interactive view of their PRs without any manual copy-paste.

## Background

The two-phase `continue_transaction` approach was added to let agents display PR links before blocking, but it added complexity and broke the natural flow. The simpler model is: `await_merge` opens PRs, optionally opens them in the browser (so the user sees them immediately), then blocks polling until all are merged. The agent doesn't need to take a second action — the browser opening handles the UX need.

## Design

### 1. Revert `continue_transaction` (ops/contract.go, ops/ship.go, mcp/src/tools/ship.ts)

Remove `ContinueTransaction bool` from `ShipInput`. Remove the `continue_transaction` parameter from the MCP tool schema. Remove the `awaitAndTransition` helper and Phase 1/2 split from `ops/ship.go`.

Replace the `await_merge` case with an inline infinite loop:

```go
case config.PRModeAwaitMerge:
    plan, err := resolve.Build(l)
    if err != nil {
        return "", err
    }
    const pollInterval = 30 * time.Second
    // Initial sleep — gives GitHub time to index the newly opened PRs.
    time.Sleep(pollInterval)
    for {
        open := 0
        for _, repo := range plan.Repos {
            open += len(github.ListOpenPRs(repo.Dest))
        }
        if open == 0 {
            nextBranch := namegen.Generate()
            if err := checkout.Run(l, checkout.Options{
                RequireMergedPR: false,
                NextBranch:      nextBranch,
            }); err != nil {
                return "", err
            }
            return fmt.Sprintf("\nmerged: all PRs merged → transitioned to %s", nextBranch), nil
        }
        time.Sleep(pollInterval)
    }
```

### 2. `open_browser` config flag (internal/config/config.go)

Add to `ShipSetup`:

```go
OpenBrowser bool `yaml:"open_browser"`
```

Example `config.yaml`:
```yaml
ship:
  pr_mode: await_merge
  open_browser: true
  delete_branch_on_merge: true
```

### 3. Open browser tabs in ops/ship.go

After `ship.Run(...)` succeeds and before entering the `switch` on `PRMode`, if `l.Setup.Ship.OpenBrowser` is true, collect all open PR URLs on the deck and open each in the default browser:

```go
if l.Setup.Ship.OpenBrowser {
    cmd := "open"       // macOS
    if runtime.GOOS != "darwin" {
        cmd = "xdg-open"  // Linux
    }
    plan, _ := resolve.Build(l)
    for _, repo := range plan.Repos {
        for _, pr := range github.ListOpenPRs(repo.Dest) {
            exec.Command(cmd, pr.URL).Start()
        }
    }
}
```

Uses `runtime`, `os/exec` imports. Errors are silently ignored — browser opening is best-effort.

### 4. MCP tool hint update (mcp/src/tools/ship.ts)

Remove `continue_transaction` from schema and handler. Update the `awaitMerge` hint to:
```
PRs are open and waiting to be merged. hv_ship is polling — it will transition automatically once all PRs are merged.
```

## Implementation phases

| Phase | Title             | Description                                                                                 |
|-------|-------------------|---------------------------------------------------------------------------------------------|
| 0000  | Status check      | `hv_status hive-deck-pro` — clean before starting                                          |
| 0001  | Revert two-phase  | Remove `ContinueTransaction` from contract.go, ship.go, ship.ts; restore inline loop       |
| 0002  | open_browser flag | Add `OpenBrowser` to config; open tabs in ops/ship.go after ship.Run succeeds               |
| 0003  | Build + ship      | `go build`, `go install`, `npm run build`; ship hive-deck-pro                              |
