# Plan: pr_mode Enum — Replace auto_merge + require_merged_pr

## Objective

Replace the two boolean ship config fields (`auto_merge` and `require_merged_pr`) with a single `pr_mode` enum that makes PR handling intent explicit. Three modes: `auto_merge` (current behaviour — GitHub merges immediately, then auto-transition), `await_merge` (open PRs, poll until all merged, then auto-transition), and `manual` (open PRs and stop — user merges and runs `hv next` themselves). A new `hv_await_merge` MCP tool is added for the `await_merge` mode with a descriptor that instructs the agent to run it as a background `/loop 30s` task.

## Background

The current config has two booleans that together encode three distinct intents:

| `auto_merge` | `require_merged_pr` | Actual intent |
|---|---|---|
| `true` | `true` | Merge immediately, auto-transition |
| `false` | `true` | Open PRs, block on user, manual next |
| `false` | `false` | Open PRs, transition immediately (skip merge wait) |

This is ambiguous and doesn't accommodate the new `await_merge` intent cleanly. A single `pr_mode` enum expresses all three intents directly, removes the boolean interaction surface, and enables the polling path without adding a fourth boolean.

Migration is lossless: `auto_merge: true` → `pr_mode: auto_merge`; `auto_merge: false, require_merged_pr: true` → `pr_mode: manual`; `auto_merge: false, require_merged_pr: false` → `pr_mode: auto_merge` (closest equivalent — immediate transition).

Existing deck configs (`~/.hv/decks/*.yaml`) don't set `ship:` at all — they inherit from `config.yaml`. Only `config.yaml` needs updating on each machine.

## Design

### Config struct change

```go
// internal/config/config.go — Ship struct

// Before:
type Ship struct {
    AutoMerge           bool `yaml:"auto_merge"`
    RequireMergedPR     bool `yaml:"require_merged_pr"`
    DeleteBranchOnMerge bool `yaml:"delete_branch_on_merge"`
    TeardownOnShip      bool `yaml:"teardown_on_ship"`
    AutoDefaultBranch   bool `yaml:"auto_default_branch"`
}

// After:
type PRMode string

const (
    PRModeAutoMerge  PRMode = "auto_merge"
    PRModeAwaitMerge PRMode = "await_merge"
    PRModeManual     PRMode = "manual"
)

type Ship struct {
    PRMode              PRMode        `yaml:"pr_mode"`
    DeleteBranchOnMerge bool          `yaml:"delete_branch_on_merge"`
    TeardownOnShip      bool          `yaml:"teardown_on_ship"`
    AutoDefaultBranch   bool          `yaml:"auto_default_branch"`
    MergePollInterval   time.Duration `yaml:"merge_poll_interval"` // default 30s
    MergePollTimeout    time.Duration `yaml:"merge_poll_timeout"`  // default 30m
}
```

### Backward-compatible YAML unmarshalling

Add a custom `UnmarshalYAML` on `Ship` that reads old `auto_merge`/`require_merged_pr` booleans and maps them to `PRMode` if `pr_mode` is absent:

```go
func (s *Ship) UnmarshalYAML(value *yaml.Node) error {
    // decode into a raw map, check for pr_mode first
    // if absent, check auto_merge + require_merged_pr and derive pr_mode
    // this makes the migration zero-downtime — old configs still work
}
```

### ops/ship.go — dispatch on pr_mode

```go
switch l.Setup.Ship.PRMode {
case config.PRModeAutoMerge:
    // current behavior: auto-merge flag on PR, then checkout next branch
case config.PRModeAwaitMerge:
    // open PRs normally, return — caller polls via RunAwaitMerge
case config.PRModeManual:
    // open PRs, print merge-gate message, return
default:
    // treat missing/empty as auto_merge for safety
}
```

### ops/await_merge.go — new file

```go
type AwaitMergeInput struct {
    Op   string `json:"op"`
    Deck string `json:"deck"`
}

func RunAwaitMerge(in AwaitMergeInput) (string, error) {
    l, err := config.LoadDeck(in.Deck)
    interval := l.Setup.Ship.MergePollInterval
    if interval == 0 { interval = 30 * time.Second }
    timeout := l.Setup.Ship.MergePollTimeout
    if timeout == 0 { timeout = 30 * time.Minute }

    deadline := time.Now().Add(timeout)
    for time.Now().Before(deadline) {
        plan, _ := resolve.Build(l)
        open := countOpenPRs(plan.Repos)
        if open == 0 {
            nextBranch := namegen.Generate()
            err := checkout.Run(l, checkout.Options{
                RequireMergedPR: false,
                NextBranch:      nextBranch,
            })
            if err != nil { return "", err }
            return fmt.Sprintf("merged: all PRs merged → transitioned to %s", nextBranch), nil
        }
        fmt.Printf("pending: %d PR(s) open — checking again in %s\n", open, interval)
        time.Sleep(interval)
    }
    return "", fmt.Errorf("timeout after %s waiting for PRs to merge", timeout)
}
```

### dispatch.go addition

```go
case "await_merge":
    var in AwaitMergeInput
    json.Unmarshal(payload, &in)
    return RunAwaitMerge(in)
```

### ops/contract updates

Add `AwaitMergeInput` to `contract.go` and document in `CONTRACT.md` + `schema.json`.

### MCP — new hv_await_merge tool

```typescript
server.tool(
  "hv_await_merge",
  `Poll until all open PRs on the deck are merged, then auto-transition to the next branch.
Use this after hv_ship when pr_mode is await_merge.

BACKGROUND TASK: After calling hv_ship, immediately register this as a background loop:
  /loop 30s check if PRs are merged on deck "<deck>" using hv_await_merge, stop when done
Do not block the conversation waiting for this — schedule it and continue.
Returns "pending: N PR(s) open" on each tick until merged.
Returns "merged: all PRs merged → transitioned to <branch>" when complete.`,
  {
    deck: z.string().describe(
      "Deck name (e.g. 'cloud-manager'). Use hv_decks to list available decks."
    ),
  },
  async ({ deck }) => {
    const { ok: success, output } = runHv({ op: "await_merge", deck });
    if (!success) return err(output);
    const done = output.startsWith("merged:");
    return ok(output, done
      ? "All PRs merged and branch transitioned. Loop complete — stop the background task."
      : "PRs still open. Loop will check again shortly."
    );
  }
)
```

### config.yaml migration

```yaml
# Before:
ship:
  auto_merge: true
  require_merged_pr: true

# After:
ship:
  pr_mode: auto_merge        # auto_merge | await_merge | manual
  delete_branch_on_merge: true
```

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000 | Setup | hv_status + hv_next |
| 0001 | Config struct | Replace booleans with `PRMode` enum + custom UnmarshalYAML for backward compat + `MergePollInterval`/`MergePollTimeout` fields |
| 0002 | ops/ship.go dispatch | Switch on `PRMode` in `RunShip`; `await_merge` opens PRs and returns without blocking |
| 0003 | ops/await_merge.go | New `RunAwaitMerge` — polling loop + checkout on completion; wire into dispatch + contract |
| 0004 | MCP hv_await_merge | New tool in `mcp/src/tools/await_merge.ts` with background-task descriptor; rebuild MCP |
| 0005 | Update configs + ship | Update `~/.hv/config.yaml` and local `.hv` copy; run tests; ship hive-deck-pro |
