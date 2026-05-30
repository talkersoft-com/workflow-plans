# Plan: Per-deck branch field

## Objective

Replace the global `default_target_branch` / per-repo `target_branches` override map in `config.yaml` with a single `branch:` field at the top of each deck YAML. Every repo in a deck uses the same branch for cloning, PR targeting, and feature-branch basing. When the configured branch does not yet exist on a remote, `hv init` creates it by branching from the repo's git-detected true default; `hv ship` ensures it exists before opening a PR. The result is one setting, one deck, consistent behaviour — and a simpler data model with no per-repo special cases.

## Background

The original design put branch overrides in `config.yaml` (machine-local) because repos within a deck sometimes had different default branches (`main`, `master`, `dev`). This created inconsistency: the override map grew, callers duplicated `BranchFor(repo, setup)` logic, and the PR base branch was determined by `git symbolic-ref refs/remotes/origin/HEAD` rather than config, so PRs silently targeted the wrong branch. The whole point of hive-deck is consistency across a deck — one branch per deck is the correct model.

## Design

### Data model

```yaml
# config.yaml — BEFORE
default_target_branch: hive
target_branches:
  vorch-lib: hive
  vorch-cli: hive

# config.yaml — AFTER
# (these keys removed entirely)
```

```yaml
# cloud-manager.yaml — BEFORE
deck:
  vm-infra/cloud-manager:
    repos:
      - talkersoft-com/cloud-manager-api

# cloud-manager.yaml — AFTER
branch: hive

deck:
  vm-infra/cloud-manager:
    repos:
      - talkersoft-com/cloud-manager-api
```

```go
// internal/config — BEFORE
type Setup struct {
    DefaultTargetBranch string            `yaml:"default_target_branch"`
    TargetBranches      map[string]string `yaml:"target_branches"`
    // ...
}

type DeckFile struct {
    Deck TreeNode `yaml:"deck"`
    // (no branch field)
}

// internal/config — AFTER
type Setup struct {
    // DefaultTargetBranch and TargetBranches removed
    // ...
}

type DeckFile struct {
    Branch string   `yaml:"branch"` // "" means use git's true remote default
    Deck   TreeNode `yaml:"deck"`
}
```

### Branch resolution

`resolve.Build` sets every `RepoPlan.Branch` from `l.DeckFile.Branch`. The `branchFor(repo, setup)` / `BranchFor` functions are deleted. If `DeckFile.Branch` is empty, each repo falls back to its git-detected true default at clone time (existing provision behaviour for missing branches already handles this path).

### init behaviour

1. For each repo, attempt `git clone --branch <deck.Branch> <url> <dest>`.
2. If `deck.Branch` is empty or does not exist on remote, provision already falls back to `gh.DefaultBranch(slug)` and clones from there.
3. **New**: after the fallback clone, if `deck.Branch` is non-empty, create it locally (`git checkout -b <deck.Branch>`) and push it to origin (`git push origin <deck.Branch>:refs/heads/<deck.Branch>`).
4. `branch.createAll` then creates the feature branch from `origin/<deck.Branch>` (fetch first).

### ship behaviour

Before `gh pr create --base <deck.Branch>`:
1. `git ls-remote --heads origin refs/heads/<deck.Branch>` — if non-empty, no-op.
2. If empty: `git push origin refs/remotes/origin/HEAD:refs/heads/<deck.Branch>` (creates from remote default when local branch absent) or `git push origin <deck.Branch>:refs/heads/<deck.Branch>` (uses local branch when present).

The `commitsAhead` check runs after `EnsureRemoteBranch` so `origin/<deck.Branch>` always resolves.

### Pre-flight (ship)

The "refuse to ship from target branch" check compares the current branch against `l.DeckFile.Branch`. If `DeckFile.Branch` is empty, compare against `defaultBranch(dir)` (git-detected) as today.

### next / branch.createAll

`createAll` receives the deck branch via the plan (`plan.Branch` — a new field on `Plan`). Feature branches are created from `origin/<plan.Branch>` after fetch. Removes the local `defaultBranch(dir)` call.

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000 | Setup | hv_status check, already on oaty-gibbon |
| 0001 | Config & resolve | Remove Setup fields; add DeckFile.Branch; update resolve.Build to populate RepoPlan.Branch and Plan.Branch from deck; delete BranchFor |
| 0002 | provision & init | Update provision fallback to push deck branch after fallback clone; update ops/init.go to call EnsureRemoteBranch per repo |
| 0003 | branch & next | Update branch.createAll to use plan.Branch instead of defaultBranch(dir) |
| 0004 | ship | Use deck branch as PR base; move EnsureRemoteBranch before commitsAhead; fix pre-flight check |
| 0005 | YAML updates | Add `branch: hive` to all deck YAMLs; remove dead keys from config.yaml |
| 0006 | Cleanup & build | Delete BranchFor, defaultBranch in branch.go, unused Setup fields; go build; make install |
