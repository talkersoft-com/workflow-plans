# Plan: Go test suite for checkout package

## Objective

Add `internal/checkout/checkout_test.go` as the first Go test file in the codebase. The tests exercise the rollback behavior added in the previous PR using real temporary git repos — no mocks, no third-party frameworks.

## Background

`internal/checkout/checkout.go` has no test coverage. The rollback logic (`savedBranch` map, `switched` slice, `rollback()` closure, `gitCheckout` helper) was added without tests. Since `Run()` requires a full loaded config/deck, the most practical entry point is to test the internal helpers directly by calling them from within the same package (`package checkout`), using `t.TempDir()` git repos as fixtures.

## Design

**Package**: `package checkout` (white-box — same package as the code under test, so unexported helpers are accessible directly).

**Test helpers**: a `makeRepo(t, branches...)` helper that creates a temp git repo, makes an initial commit, and creates all named branches. Returns the repo path.

**Test cases**:

### `TestGitCheckout_SwitchesBranch`
- Create a temp repo with branches `main` and `feature`
- Verify starting on `main`
- Call `gitCheckout(dir, "feature")`
- Assert active branch is now `feature`
- Call `gitCheckout(dir, "main")`
- Assert active branch is back to `main`

### `TestRollback_RestoresPreviousBranch`
- Create two temp repos (`repoA`, `repoB`), both starting on `main` with a `feature` branch
- Simulate Phase 2 partial success: switch `repoA` to `feature` (appended to `switched`), `repoB` fails
- Build `savedBranch` map: both repos → `"main"`
- Invoke the rollback loop directly (iterate `switched` in reverse, call `gitCheckout` with saved branch)
- Assert `repoA` is back on `main`

**No test for `Run()` itself** — it requires config/deck loading. That's a future phase.

## Implementation phases

| Phase | Title  | Description                                                               |
|-------|--------|---------------------------------------------------------------------------|
| 0000  | Setup  | hv_status + hv_next                                                       |
| 0001  | Tests  | Write `internal/checkout/checkout_test.go` with `makeRepo` helper and both test cases; `go test ./internal/checkout/...` must pass |

## Open questions

None.
