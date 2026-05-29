# Plan: rename target branch config + smart init branch resolution

## Objective

Two improvements to hive-deck: (1) rename `default_branch` / `branches:` to `default_target_branch` / `target_branches:` to make clear these are PR target branches, not git's concept of a default branch; (2) make `hv init` smart about branch existence — if the target branch doesn't exist on the remote, auto-detect the repo's true GitHub default branch and create the target branch from it, eliminating the need for manual `target_branches:` overrides.

## Background

The current `default_branch` / `branches:` naming causes confusion with git's own default branch concept. They are really "the branch we target for PRs and base new work from" — not necessarily the repo's GitHub default. Separately, `hv init` hard-fails when a repo's target branch doesn't exist on the remote (e.g. `vmorchestrator` uses `master` not `main`), forcing users to manually add per-repo overrides to `config.yaml`. The smart resolution makes the tool self-healing.

## Design

### Phase 1 — Config rename

**`internal/config/config.go`**:
```go
// Old fields (keep for backwards compat, read with deprecation warning)
// New fields:
DefaultTargetBranch string            `yaml:"default_target_branch"`
TargetBranches      map[string]string `yaml:"target_branches"`
```

Add `UnmarshalYAML` or post-load logic: if `default_target_branch` is empty but `default_branch` is set, use `default_branch` and print:
```
warning: config.yaml: 'default_branch' is deprecated — rename to 'default_target_branch'
```
Same for `branches:` → `target_branches:`.

Update `config.yaml.example` to use the new keys.

### Phase 2 — Smart branch resolution in provision

**`internal/github/github.go`** — add:
```go
// DefaultBranch returns the true GitHub default branch for a repo slug.
// Returns "main" on any error (safe fallback).
func DefaultBranch(slug string) string {
    cmd := exec.Command("gh", "repo", "view", slug, "--json", "defaultBranchRef", "--jq", ".defaultBranchRef.name")
    out, err := cmd.Output()
    if err != nil || strings.TrimSpace(string(out)) == "" {
        return "main"
    }
    return strings.TrimSpace(string(out))
}
```

**`internal/provision/provision.go`** (or wherever clone happens) — replace the current clone-at-branch logic with:
```go
// 1. Check if target branch exists on remote
exists := remoteBranchExists(repoURL, targetBranch) // git ls-remote --heads <url> <branch>

if exists {
    // Current behavior: clone at target branch
    git clone --branch <targetBranch> <url> <dest>
} else {
    // New behavior: find true default, clone there, create target branch
    truDefault := github.DefaultBranch(slug)
    git clone --branch <trueDefault> <url> <dest>
    git checkout -b <targetBranch>  // create target branch from true default
}
```

`remoteBranchExists` uses `git ls-remote --heads <url> refs/heads/<branch>` — non-empty output means the branch exists.

## Implementation phases

| Phase | Title      | Description                                                                                     |
|-------|------------|-------------------------------------------------------------------------------------------------|
| 0000  | Setup      | hv_status + hv_next                                                                             |
| 0001  | Rename     | Rename config fields in config.go + config.yaml.example; backwards compat with deprecation warning |
| 0002  | Smart init | Add `DefaultBranch()` to github.go; update provision to check remote branch existence before cloning |
| 0003  | Build+Test | `make install`; test against samples deck — `vmorchestrator` should init without a `target_branches:` override |

## Open questions

None.
