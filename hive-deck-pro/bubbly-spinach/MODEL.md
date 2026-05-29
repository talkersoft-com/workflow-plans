# Model: Built-in + Custom seam

This file shows the exact shape of the internal YAML and fragments, and where
custom user content slots in. Every file shown here is a real file that will
exist in `internal/builtin/` inside the Go project.

---

## 1. The built-in base YAML

`internal/builtin/workflow/_base.yaml`

```yaml
type: workflow

init:
  fragments:
    - "@status-check"
    - "@folder-structure"
    - "@write-orch"
    - "@write-task-0000"
    - "@write-test-0000"

# ── CUSTOM PHASES SLOT ──────────────────────────────────────────────────────
# At assembly time, each user-defined phase is expanded here.
# If no extension is given, this slot is empty and only init + post run.
# Each phase becomes one TASK file + one TEST file using @write-task / @write-test.
custom_phases: "{{.Phases}}"
# ────────────────────────────────────────────────────────────────────────────

post:
  fragments:
    - "@write-result-lessons"
    - "@key-rules"
    - "@pr-mode-instructions"
```

---

## 2. What the assembler does with `{{.Phases}}`

When a user extension is loaded, `{{.Phases}}` expands to one rendered block
per phase, each block injecting `{{.PhaseTitle}}` and `{{.Jobs}}` into the
`@write-task` and `@write-test` fragments:

```
{{.Phases}} →

  phase[0]:  title="Database"       jobs=["@check-database", "@update-ef", "@deploy-ef"]
  phase[1]:  title="Infrastructure" jobs=["@run-pipeline-mcp", "@modify-glue-jobs"]
  phase[2]:  title="Migration"      jobs=["@cdk-synth", "@run-conversion"]
```

When NO extension is given, `{{.Phases}}` is an empty slice — slot is skipped entirely.

---

## 3. The fragment that renders each phase

`internal/builtin/workflow/fragments/write-task.md`

```markdown
## Task {{.TaskNumber}} — {{.PhaseTitle}}

{{range .Jobs}}
### {{.}}

{{fragment .}}
{{end}}

## Definition of done
All jobs above complete without error.
```

`{{.TaskNumber}}` is injected by the assembler (0001, 0002, ...).
`{{range .Jobs}}` iterates the jobs array for this phase.
`{{fragment .}}` resolves the fragment ref (e.g. `@check-database`) from disk or binary.

---

## 4. A user extension YAML (on disk, not in binary)

`~/.hv/workflows/workflow/db-migration.yaml`

```yaml
tokens:
  MigrationTool: "{{.DeckRoot}}/tools/migration"

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

---

## 5. The fully assembled output (what the agent sees)

```
━━━ Task 0000 — Init ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  ← base.init fragments
  [contents of @status-check]
  [contents of @folder-structure]
  [contents of @write-orch]
  [contents of @write-task-0000]
  [contents of @write-test-0000]

━━━ Task 0001 — Database ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  ← user phase[0]
  [contents of @check-database]                             jobs expanded inline
  [contents of @update-entity-framework]
  [contents of @deploy-ef]

━━━ Task 0002 — Infrastructure ━━━━━━━━━━━━━━━━━━━━━━━  ← user phase[1]
  [contents of @run-pipeline-mcp]
  [contents of @modify-glue-jobs]

━━━ Task 0003 — Migration ━━━━━━━━━━━━━━━━━━━━━━━━━━━━  ← user phase[2]
  [contents of @cdk-synth]
  [contents of @run-conversion]

━━━ Task FINAL ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  ← base.post fragments
  [contents of @write-result-lessons]
  [contents of @key-rules]
  [contents of @pr-mode-instructions]
```

---

## 6. Base-only mode (no extension)

```
hv workflow hive-deck-pro          # no name = no extension
```

```
━━━ Task 0000 — Init ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  ← base.init
  [status-check, folder-structure, write-orch, ...]

                                                         ← {{.Phases}} = [] → nothing

━━━ Task FINAL ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  ← base.post
  [write-result-lessons, key-rules, ...]
```

Valid for a simple change with no structured phases needed.

---

## 7. How changing the built-in affects custom workflows

Because custom workflows only define `phases[]` and `tokens:`, changes to the
built-in base automatically propagate to every custom workflow:

| Change to built-in | Effect on custom workflows |
|--------------------|---------------------------|
| Add fragment to `init` | Every workflow gets it in Task 0000 |
| Add fragment to `post` | Every workflow gets it in Task FINAL |
| Rename a token | Must update fragments that reference it — custom workflows inherit the fix |
| Change `@write-task` template | All phase rendering changes for everyone |

Custom workflows are insulated from built-in changes to init/post — they never
define those sections, so there is no conflict. The seam is clean.
