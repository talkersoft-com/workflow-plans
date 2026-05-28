# Plan: hv_ship await_merge two-phase transaction

## Objective

`hv_ship` with `pr_mode: await_merge` uses a two-call transaction so the agent can display clickable PR links to the user before blocking to poll. Phase 1 opens PRs and returns immediately with links and instructions. Phase 2 (triggered by `continue_transaction: true`) polls indefinitely until all PRs are merged, then transitions the deck.

## Background

The previous single-call design either polled infinitely or timed out inside one MCP call — the agent never got a chance to surface PR links to the user. The fix is to split the work: Phase 1 is fast (open PRs, return links), Phase 2 is the blocking poll.

## Design

### ShipInput — new optional field

```go
type ShipInput struct {
    Op                  string `json:"op"`
    Deck                string `json:"deck"`
    Message             string `json:"message"`
    Title               string `json:"title"`
    Body                string `json:"body"`
    ContinueTransaction bool   `json:"continue_transaction,omitempty"`
}
```

Field description for MCP schema (in `dispatch.go`):
> Optional. Set to true to resume an await_merge ship and poll until all PRs are merged and the deck transitions. Only valid after an initial hv_ship call that already opened PRs.

### Phase 1 — `RunShip` (no `continue_transaction`, `await_merge` mode)

1. Commit, push, open PRs as today
2. Collect the PR URLs returned by the ship step
3. Return immediately with:
```
PRs opened — review and merge when ready:
  <repo>   <pr-url>
  ...

[await_merge] Call hv_ship again with continue_transaction=true to poll until all PRs are merged and transition to the next branch.
```

### Phase 2 — `RunShip` (`continue_transaction: true`)

1. Skip commit/push/PR-open entirely
2. Call `resolve.Build` to get the repo list
3. Poll `github.ListOpenPRs` every 30 seconds indefinitely
4. On first poll iteration, sleep 30s first (avoids false-zero if GitHub hasn't indexed yet)
5. When open count reaches 0: call `checkout.Run` to transition, return:
```
merged: all PRs merged → transitioned to <branch>
```

### dispatch.go

Register `continue_transaction` in the MCP input schema with `omitempty` so it only appears in the tool description when the agent needs it, reducing noise on normal calls.

## Implementation phases

| Phase | Title                  | Description                                                                 |
|-------|------------------------|-----------------------------------------------------------------------------|
| 0000  | Status check           | `hv_status hive-deck-pro` — clean before starting                          |
| 0001  | ShipInput + dispatch   | Add `continue_transaction` field to `ShipInput`; register in `dispatch.go` |
| 0002  | Phase 1 logic          | `await_merge` case: open PRs, collect URLs, return with agent instructions  |
| 0003  | Phase 2 logic          | `continue_transaction: true` case: infinite poll, transition on zero        |
| 0004  | Integration test       | Ship a README change, verify Phase 1 returns links, Phase 2 blocks then transitions |

## Constraints

- `continue_transaction` is ignored for `auto_merge` and `manual` modes
- Phase 2 must not re-commit or re-open PRs
- No timeout on the Phase 2 poll loop
