# Plan: auto_merge polls until merged before transitioning

## Objective

Change `auto_merge` mode in `ops/ship.go` so hv_ship opens PRs with GitHub auto-merge enabled, then polls until all PRs are actually merged before transitioning to the next branch. If any PR is closed without merging, the deck stays on the current branch and an error is returned — no silent partial state.

## Background

Currently `auto_merge` commits, pushes, opens PRs with GitHub auto-merge, then immediately calls `checkout.Run()` to transition locally. The PRs are merged by GitHub in the background, but by then the deck has already moved on. This means:
- The user can't know if a PR failed to auto-merge until they check GitHub manually
- The deck has transitioned even if CI blocked the merge

`await_merge` already has polling logic in `ops/await_merge.go`. `auto_merge` should behave the same way — poll until done — but with GitHub doing the merging automatically rather than the user.

## Design

**Extract a shared poll function from `ops/await_merge.go`:**

```go
// PollUntilMerged polls the open PRs for the deck until all are merged or one fails.
// Returns nil when all merged, error if any PR is closed without merging.
func PollUntilMerged(l *config.Loaded, interval time.Duration) error
```

This function already exists conceptually inside `ops/await_merge.go` — extract it so both `await_merge` and `auto_merge` can call it.

**Change `auto_merge` case in `ops/ship.go`:**

```go
case config.PRModeAutoMerge, "":
    // Poll until all PRs merge (GitHub auto-merge does the work).
    if err := awaitmerge.PollUntilMerged(l, 5*time.Second); err != nil {
        return "", err  // deck stays on current branch
    }
    // All merged — now transition.
    if l.Setup.Ship.TeardownOnShip {
        return "", teardown.Run(l, teardown.Options{RequireMergedPR: false})
    }
    nextBranch := namegen.Generate()
    return "", checkout.Run(l, checkout.Options{
        RequireMergedPR: false,
        NextBranch:      nextBranch,
    })
```

**Failure condition:** PR state == `CLOSED` (not `MERGED`). Still `OPEN` = still pending, keep polling. The poll interval should match `await_merge` (5s). Print periodic status so the user sees progress.

## Implementation phases

| Phase | Title       | Description                                                                                          |
|-------|-------------|------------------------------------------------------------------------------------------------------|
| 0000  | Setup       | hv_status + hv_next                                                                                  |
| 0001  | Extract     | Extract `PollUntilMerged` from `ops/await_merge.go` into a shared function callable by both modes   |
| 0002  | Wire        | Update `auto_merge` case in `ops/ship.go` to call `PollUntilMerged` before `checkout.Run()`         |
| 0003  | Build+Ship  | `make install`, write Results + Retro/LESSONS, hv_ship                                              |

## Open questions

None.
