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

### Go model shapes

**Built-in base** (`internal/builtin/<type>/_base.yaml` → loaded into `BuiltinBase`):
```go
// BuiltinBase is embedded in the binary for both "workflow" and "plan" types.
// Init and Post are the fixed bookends — the user never touches these.
type BuiltinBase struct {
    Type string      `yaml:"type"` // "workflow" | "plan"
    Init BaseSection `yaml:"init"` // always Task 0000
    Post BaseSection `yaml:"post"` // always Task FINAL
}

type BaseSection struct {
    Fragments []string `yaml:"fragments"` // e.g. ["@status-check", "@write-task-0000"]
}
```

**User extension — workflow** (`~/.hv/workflows/workflow/<name>.yaml`):
```go
// WorkflowExtension is what the user defines. It fills the slot between
// base.Init and base.Post. Tokens layer on top of the built-in token set.
type WorkflowExtension struct {
    Tokens map[string]string `yaml:"tokens"` // optional custom tokens
    Phases []Phase           `yaml:"phases"` // one TASK file per phase
}

type Phase struct {
    Title string   `yaml:"title"`
    Jobs  []string `yaml:"jobs"` // fragment refs, e.g. "@check-database"
}
```

**User extension — plan** (`~/.hv/workflows/plan/<name>.yaml`):
```go
// PlanExtension appends additional fragments after the base plan fragments.
// Simpler than workflow: no phase/job hierarchy, just extra steps.
type PlanExtension struct {
    Tokens    map[string]string `yaml:"tokens"`
    Fragments []string          `yaml:"fragments"`
}
```

**Assembled result** (internal, passed to renderer):
```go
// AssembledWorkflow is the merged output of base + extension, ready to render.
type AssembledWorkflow struct {
    Type   string            // "workflow" | "plan"
    Tokens map[string]string // built-in tokens merged with user custom tokens
    Tasks  []AssembledTask
}

type AssembledTask struct {
    Number    string   // "0000", "0001", ..., "FINAL"
    Title     string
    Source    string   // "builtin" | "user"
    Fragments []string // resolved fragment content for this task
}
```

---

### Where the user extension slots in

```
BuiltinBase.Init  ─────────────────────────────────  Task 0000  (built-in, always)
                         ▲
WorkflowExtension.Phases ┤  phases[0]  ──────────────  Task 0001  (user)
                         │  phases[1]  ──────────────  Task 0002  (user)
                         │  phases[N]  ──────────────  Task 000N  (user)
                         ▼
BuiltinBase.Post  ─────────────────────────────────  Task FINAL  (built-in, always)
```

The base has no middle. The slot between Init and Post is empty unless an extension
fills it. Base-only mode (`hv workflow <deck>` with no name) produces just Task 0000
and Task FINAL — a valid minimal workflow for simple changes.

---

### Merge mechanism in RunWorkflow

```
RunWorkflow(type, deck, name?)
  │
  ├─ 1. Load BuiltinBase from embed.FS
  │       binary://internal/builtin/<type>/_base.yaml
  │
  ├─ 2. Build built-in token set
  │       Deck, Branch, DeckRoot, PlanFolder, ExecFolder,
  │       PRMode, Agent, DeleteBranchOnMerge, OpenBrowser
  │
  ├─ 3. If name given → load extension from disk
  │       ~/.hv/workflows/<type>/<name>.yaml
  │       Validate: workflow→phases not empty, each phase has title+jobs
  │                 plan→fragments not empty
  │       Merge tokens: built-in set + extension.Tokens
  │       (user tokens can reference built-in tokens: "{{.DeckRoot}}/tools/foo")
  │
  ├─ 4. Assemble task list
  │       Task 0000  ← base.Init.Fragments       (always first)
  │       Task 0001  ← phases[0] jobs expanded   (user, workflow only)
  │       Task 000N  ← phases[N] jobs expanded   (user, workflow only)
  │       Task FINAL ← base.Post.Fragments       (always last)
  │
  └─ 5. Render: resolve all {{.Token}} across all fragment content
          → assembled prompt string returned to caller
```

For `plan` type, step 4 is flat: `base.Fragments + extension.Fragments` in order.
No task splitting — plan assembly produces a single prompt, not a task file per phase.

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
