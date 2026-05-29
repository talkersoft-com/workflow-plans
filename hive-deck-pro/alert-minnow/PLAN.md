# Plan: Embed built-in workflows + optional name + token cleanup

## Objective
Move the `plan` and `workflow` bases out of the `workflow-fragments` repo and into the
`hive-deck-pro` binary via `//go:embed`, making them always available with zero external
config. Workflow name becomes optional (no name = base only). Token names are simplified
(`WorkflowPlanningFolder` → `PlanFolder`, `WorkflowFolder` → `ExecFolder`). MCP tools
are updated to mirror the new CLI. The result is a self-bootstrapping binary — a fresh
machine with only `hv` installed can run `hv workflow plan <deck>` or `hv workflow <deck>`
immediately, with no repo provisioning required.

## Background
Currently `hv workflow <deck> <name>` requires the `workflow-fragments` repo to be
provisioned on disk. The `planning.yaml` and `create-workflow.yaml` files live per-deck
inside that repo and have already diverged between `hive-deck-pro` and `cloud-manager`.
This is the same chicken-and-egg problem as a database catalog: the tool that provisions
repos cannot itself be configured until repos are provisioned. The fix is to embed the
catalog — the two foundational workflow types — directly into the binary, exactly as a
database hard-codes its system tables.

The `workflow-fragments` repo is NOT going away. It remains the home for user-facing
custom fragments and named extension YAMLs. It just stops being load-bearing for
bootstrap.

## Design

### CLI
```
hv workflow plan <deck> [name]     # plan base (+ named extension if given)
hv workflow <deck> [name]          # workflow base (+ named extension if given)
hv workflow list <deck>            # shows built-ins + user-defined, tagged
```
`plan` is a fixed subcommand, not a workflow name. Name is optional in both forms.

### Binary layout
```
internal/builtin/
  plan/
    _base.yaml          ← assembled when no name given
    fragments/
      write-plan.md
      write-deck.md
      write-init.md
      ship-plans.md
      plan-overview.md
      post-ship-reminder.md
  workflow/
    _base.yaml          ← assembled when no name given
    fragments/
      folder-structure.md
      write-orch.md
      write-task-0000.md
      write-task.md
      write-test-0000.md
      write-test.md
      write-result-lessons.md
      key-rules.md
      status-check.md
      pr-mode-instructions.md
      rebuild-mcp-artifacts.md
```

### Resolution order (RunWorkflow)
1. Load `_base.yaml` from binary (always)
2. If name given: load `~/.hv/workflows/<type>/<name>.yaml` from disk
3. Assemble: base fragments + extension fragments, in order
4. Resolve all tokens against the merged token set

### Token changes
| Old | New | Notes |
|-----|-----|-------|
| `{{.WorkflowPlanningFolder}}` | `{{.PlanFolder}}` | Derived: `DeckRoot/planning/workflow-plans` |
| `{{.WorkflowFolder}}` | `{{.ExecFolder}}` | Derived: `DeckRoot/<workflow-name-or-deck>` |

Both derived automatically from `DeckRoot` — removed as configurable token keys in YAML.
All fragments updated to use new names.

### User extension layout
```
~/.hv/workflows/
  plan/<name>.yaml            # extends plan base
  workflow/<name>.yaml        # extends workflow base
  fragments/<name>.md         # custom fragments referenced by extensions
```
Extension YAML supports `tokens:` and `fragments:` keys identical to today.
Custom tokens layer on top of built-in set; reserved names documented in `hv workflow list`.

### MCP tools
| New tool | Replaces | Notes |
|----------|----------|-------|
| `hv_workflow_run(deck, name?)` | `hv_workflow` | name optional |
| `hv_workflow_plan(deck, name?)` | `hv_plan` | name optional |
| `hv_workflow_list(deck)` | `hv workflow --list` | |

### workflow-fragments cleanup
Delete:
- `workflows/hive-deck-pro/create-workflow.yaml`
- `workflows/cloud-manager/create-workflow.yaml`
- `workflows/shared/planning.yaml`

Keep: `fragments/` folder intact (user-facing custom fragments stay on disk).

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000 | Setup | hv_status + hv_next |
| 0001 | Builtin package | Create `internal/builtin/`, embed files, wire into `RunWorkflow` |
| 0002 | Optional name + plan subcommand | Update CLI cobra commands and `ops/workflow.go` |
| 0003 | Token rename | Rename tokens in render.go, all fragments, update tests |
| 0004 | MCP tools | Update `ops/contract.go`, `ops/workflow.go`, `mcp/src/` TS tools |
| 0005 | Cleanup + ship | Delete obsolete workflow-fragments YAMLs, build, test, hv_ship |

## Open questions
None — design is settled.
