# Plan: Built-in workflow embedding + phase/job hierarchy

## Objective
Restructure the `hv workflow` system so that the built-in `plan` and `workflow` bases are
embedded in the Go binary, workflow name becomes optional, and user-defined workflows are
expressed as a structured `phases[]` / `jobs[]` hierarchy rather than a flat fragment list.
The built-in always owns `init` (Task 0000) and `post` (Task FINAL). Users define only the
phases in between, each phase mapping to one TASK file, each containing an ordered array of
jobs. The result is a self-bootstrapping binary where ordering is enforced by data shape,
not fragment concatenation order.

## Background
Two problems converged into this design:

**Bootstrap / chicken-and-egg.** `hv workflow` currently requires `workflow-fragments` to
be provisioned on disk. A fresh machine cannot plan or scaffold a workflow until the repos
that define those workflows already exist. `planning.yaml` and `create-workflow.yaml` live
per-deck and have already diverged between `hive-deck-pro` and `cloud-manager`. Embedding
the bases in the binary eliminates the dependency.

**Ordering fragility.** The current model assembles a workflow by concatenating fragments in
whatever order the YAML lists them. There is no structural enforcement of what comes first,
what comes last, or what belongs in the middle. A user can accidentally omit init, place
ship before results, or produce a prompt with no coherent phases. The phase/job model makes
the shape explicit and validates it at assembly time.

## Design

### CLI
```
hv workflow plan <deck> [name]     # plan base (+ named extension if given)
hv workflow <deck> [name]          # workflow base (+ named extension if given)
hv workflow list <deck>            # tagged: built-in | user
```
`plan` is a fixed subcommand routed before the deck argument. Name is optional in both
forms. No name = base only from binary. With name = base + named extension from disk.

### Hierarchy model
```
Workflow
  init   (built-in, always Task 0000)
  phases[]
    title  string
    jobs[] string   ← fragment refs, e.g. "@check-database"
  post   (built-in, always Task FINAL)
```

Each phase produces one TASK file and one TEST file. Jobs expand inline inside the task.
A phase is independently testable and has a pass/fail boundary. A job is atomic — if it
fails, the whole phase fails together.

**Rule of thumb:** phase = independently testable unit. Job = sequential step within it.

### User extension YAML shape
```yaml
# ~/.hv/workflows/workflow/db-migration.yaml
tokens:
  MigrationTool: "{{.DeckRoot}}/tools/migration"   # optional custom tokens
phases:
  - title: "Database"
    jobs:
      - "@check-database"
      - "@update-entity-framework"
      - "@deploy-ef"
  - title: "Infrastructure"
    jobs:
      - "@run-pipeline-mcp"
      - "@modify-glue-jobs"
  - title: "Migration"
    jobs:
      - "@cdk-synth"
      - "@run-conversion"
```

Produces:
```
Task 0000  →  init             (built-in)
Task 0001  →  Database         (jobs inline)
Task 0002  →  Infrastructure   (jobs inline)
Task 0003  →  Migration        (jobs inline)
Task FINAL →  results + ship   (built-in)
```

### Binary layout
```
internal/builtin/
  plan/
    _base.yaml
    fragments/
      write-plan.md
      write-deck.md
      write-init.md
      ship-plans.md
      plan-overview.md
      post-ship-reminder.md
  workflow/
    _base.yaml
    fragments/
      folder-structure.md
      write-orch.md
      write-task-0000.md
      write-task.md          ← receives phase title + expanded jobs
      write-test-0000.md
      write-test.md
      write-result-lessons.md
      key-rules.md
      status-check.md
      pr-mode-instructions.md
      rebuild-mcp-artifacts.md
```

### Assembly in RunWorkflow
1. Load `_base.yaml` from binary
2. If name given: load `~/.hv/workflows/<type>/<name>.yaml` from disk
3. Validate: phases array present if name given; each phase has title + jobs
4. For each phase: render the task fragment with `{{.PhaseTitle}}` and `{{.Jobs}}` injected
5. Wrap with built-in init (front) and post (back)
6. Resolve all tokens: built-in set + custom tokens from extension YAML

### Token changes
| Old | New | Derivation |
|-----|-----|-----------|
| `{{.WorkflowPlanningFolder}}` | `{{.PlanFolder}}` | `DeckRoot/planning/workflow-plans` |
| `{{.WorkflowFolder}}` | `{{.ExecFolder}}` | `DeckRoot/<deck>/<branch>` |

Both derived automatically. Removed as user-configurable keys. All fragments updated.

### MCP tools
| New | Replaces | Change |
|-----|----------|--------|
| `hv_workflow_run(deck, name?)` | `hv_workflow` | name optional |
| `hv_workflow_plan(deck, name?)` | `hv_plan` | name optional, new tool |
| `hv_workflow_list(deck)` | `hv workflow --list` | returns tagged list |

### workflow-fragments cleanup
Delete from repo:
- `workflows/hive-deck-pro/create-workflow.yaml`
- `workflows/cloud-manager/create-workflow.yaml`
- `workflows/shared/planning.yaml`

`fragments/` folder stays. Repo becomes purely additive — nothing in it is required for
bootstrap. User extension YAMLs move to `~/.hv/workflows/`.

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000 | Setup | hv_status + hv_next |
| 0001 | Builtin package | `internal/builtin/` with `//go:embed`, copy core fragments in, wire `RunWorkflow` to resolve binary first |
| 0002 | Phase/job model | New `WorkflowExtension` struct with `phases[]`/`jobs[]`; update assembler to render per-phase task files; validate structure |
| 0003 | Optional name + plan subcommand | Update cobra commands; `plan` as fixed subcommand; name optional for both forms |
| 0004 | Token rename | `WorkflowPlanningFolder`→`PlanFolder`, `WorkflowFolder`→`ExecFolder` in render.go + all fragments + tests; fix PlanFolder path to resolve inside repo not deck root |
| 0005 | MCP tools | `ops/contract.go`, `ops/workflow.go`, `mcp/src/` TS — add `hv_workflow_plan`, update `hv_workflow` → `hv_workflow_run`, add `hv_workflow_list` |
| 0006 | Cleanup + ship | Delete obsolete workflow-fragments YAMLs; `make install`; end-to-end smoke test both built-in and one user extension; write results; hv_ship |

## Open questions
None — design is settled.
