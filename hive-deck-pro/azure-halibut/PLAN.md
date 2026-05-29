# Plan: Transactional rollback for hv next

## Objective

When `hv next` fails mid-transition — because a fetch or checkout fails on one repo — every repo that already switched to the new branch is automatically rolled back to its previous branch. The deck is left in a consistent state and the user gets a clear error explaining what failed and what was rolled back.

## Background

`hv next` runs in two phases. Phase 1 validates all repos are safe to transition (no uncommitted changes, no open PRs when `require_merged_pr` is set). Phase 2 fetches origin and checks out the new branch in each repo sequentially. Currently if Phase 2 fails partway through, repos that already switched stay on the new branch while the rest remain on the old one — leaving the deck in a mixed-branch state with no automatic recovery.

The fix is simple: record each repo's current branch before Phase 2 starts, and if any step fails, git-checkout every already-switched repo back to its saved branch.

## Design

**Before Phase 2:** build a `map[string]string` of `repo.Dest → currentBranch` for all repos. `currentBranch()` already exists in the file.

**During Phase 2:** track which repos have successfully switched using a `switched []string` slice (dest paths). Append to it after each successful `gitCheckoutNewFrom`.

**On failure:** iterate `switched` in reverse order, call `git checkout <savedBranch>` on each. Log each rollback with `fmt.Fprintf(os.Stderr, "  rollback %-30s → %s\n", repo, branch)`. Rollback errors are logged but do not replace the original error.

**New helper:** `gitCheckout(dir, branch string) error` — runs `git checkout <branch>` (no `-b`).

**Return:** the original failure error, unchanged.

## Implementation phases

| Phase | Title    | Description                                                  |
|-------|----------|--------------------------------------------------------------|
| 0000  | Setup    | hv_status + confirm on azure-halibut                        |
| 0001  | Rollback | Add saved-branch map, switched slice, rollback loop, and `gitCheckout` helper to `internal/checkout/checkout.go` |
| 0002  | Fragment | Update `ship-plans.md` fragment to instruct agent to always print PR link(s) to console after `hv_ship` |

## Open questions

None — the design is self-contained and touches only `internal/checkout/checkout.go`.
