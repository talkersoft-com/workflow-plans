# Plan: hv_ship await_merge — poll until merged then auto-transition

## Objective

When `pr_mode` is `await_merge`, `hv_ship` should block and poll until all open PRs are merged, then transition the deck to the next branch automatically — identical end state to `auto_merge`, but respecting manual review gates. Today it opens PRs and immediately returns a hint message, leaving the user to set up polling themselves.

## Background

`ops/ship.go` `RunShip` already branches on `pr_mode`:
- `auto_merge` (default) — opens PRs with GitHub auto-merge enabled, immediately transitions
- `await_merge` — opens PRs, returns a message telling the user to call `hv_await_merge` manually
- `manual` — opens PRs, tells user to run `hv next` after merging

`ops/await_merge.go` `RunAwaitMerge` already does the full job: counts open PRs, and if zero, generates a new branch name and calls `checkout.Run` to transition. It's designed to be called from a `/loop` because it never sleeps — it checks once and returns.

The fix is: in the `await_merge` case inside `RunShip`, call the await-merge check logic in a loop (every 30s, up to 30 min) instead of returning immediately.

## Design

### Refactor `RunAwaitMerge` to expose a shared check

Extract the "count open PRs, transition if zero" logic from `RunAwaitMerge` into a package-level helper so `RunShip` can call it without going through the MCP op layer:

```go
// in ops/await_merge.go (or internal/ship)
func checkAndTransition(l *config.Loaded, plan *resolve.Plan) (transitioned bool, nextBranch string, err error)
```

`RunAwaitMerge` becomes a thin wrapper calling this helper. `RunShip` uses the same helper in its polling loop.

### Polling loop in `RunShip` (await_merge case)

```go
case config.PRModeAwaitMerge:
    const interval = 30 * time.Second
    const timeout  = 30 * time.Minute
    deadline := time.Now().Add(timeout)
    for time.Now().Before(deadline) {
        ok, branch, err := checkAndTransition(l, plan)
        if err != nil { return "", err }
        if ok { return fmt.Sprintf("merged: all PRs merged → transitioned to %s", branch), nil }
        time.Sleep(interval)
    }
    return "", fmt.Errorf("await_merge: timed out after 30m — PRs still open; merge them and run hv next")
```

### No change to `manual` mode

`manual` stays as-is: opens PRs, returns immediately, user calls `hv next`.

### Timeout behaviour

On timeout `RunShip` returns an error (non-nil) so the MCP layer surfaces it clearly. The deck is left in its current state — open PRs, same branch — so the user can still merge and call `hv next` manually.

## Implementation phases

| Phase | Title                  | Description                                                                                  |
|-------|------------------------|----------------------------------------------------------------------------------------------|
| 0000  | Status check           | `hv_status hive-deck-pro` — clean before starting                                           |
| 0001  | Refactor await logic   | Extract `checkAndTransition` helper from `RunAwaitMerge`; existing MCP tool still passes    |
| 0002  | Polling loop in ship   | Add loop in `RunShip` `await_merge` case; build + unit tests pass                           |
| 0003  | Integration verify     | Set `pr_mode: await_merge` in config, ship a test change, confirm auto-transition on merge  |
