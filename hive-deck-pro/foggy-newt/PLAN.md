# Plan: Go template engine for workflow fragments

## Objective

Replace the `{{Key}}` string-substitution renderer in `internal/workflow/render.go` with Go's `text/template` package. Fragment `.md` files become full Go templates with access to typed config fields, enabling conditional output — e.g. different ship instructions depending on `pr_mode`.

## Background

The current `Render` function calls `strings.ReplaceAll` for each token in a flat `map[string]string`. This works for variable substitution but cannot express conditionals. The user wants fragments to branch on config values:

```
{{if eq .PRMode "await_merge"}}
PRs will be polled automatically — no action needed after ship.
{{else}}
Merge the PRs manually, then run hv_next.
{{end}}
```

The fix is a one-layer change: swap `strings.ReplaceAll` for `text/template` execution against a typed `TemplateData` struct that carries both the existing tokens and strongly-typed config fields loaded from the deck's `config.yaml`.

## Design

### TemplateData struct

```go
// TemplateData is the template context passed to every fragment.
type TemplateData struct {
    Deck                   string
    Branch                 string
    DeckRoot               string
    WorkflowFolder         string
    WorkflowPlanningFolder string
    // Ship config
    PRMode              string
    OpenBrowser         bool
    DeleteBranchOnMerge bool
}
```

Exported so external tooling can construct it. Lives in `internal/workflow/render.go`.

### Updated Render signature

```go
func Render(root string, fragments []string, data TemplateData) (string, error)
```

Implementation:
```go
tmpl, err := template.New(ref).Parse(content)
if err != nil {
    return "", fmt.Errorf("render: fragment %q: %w", ref, err)
}
var buf bytes.Buffer
if err := tmpl.Execute(&buf, data); err != nil {
    return "", fmt.Errorf("render: fragment %q: %w", ref, err)
}
parts = append(parts, buf.String())
```

### Fragment syntax change

Old: `{{Deck}}` `{{Branch}}` `{{WorkflowFolder}}`  
New: `{{.Deck}}` `{{.Branch}}` `{{.WorkflowFolder}}`

All existing fragments in `workflow-fragments/fragments/` must be migrated to dot syntax. The old bare-brace tokens will no longer be substituted — `text/template` treats them as function calls and errors.

### Caller changes

`ops/workflow.go` (or wherever `RunWorkflow` builds the token map) must:
1. Load the deck config via `config.LoadDeck`
2. Construct `TemplateData` from tokens + `l.Setup.Ship.*` fields
3. Pass `TemplateData` to `Render` instead of `map[string]string`

### Fragment migration

All `{{Key}}` references in `workflow-fragments/fragments/*.md` → `{{.Key}}`. Add a `key-rules` note documenting the new syntax and conditional pattern.

## Implementation phases

| Phase | Title                  | Description                                                                                      |
|-------|------------------------|--------------------------------------------------------------------------------------------------|
| 0000  | Status check           | `hv_status hive-deck-pro` — clean before starting                                               |
| 0001  | TemplateData + Render  | Define `TemplateData` struct; replace `strings.ReplaceAll` with `text/template` in `render.go`  |
| 0002  | Update callers         | Update `ops/workflow.go` to build `TemplateData` from deck config + tokens                       |
| 0003  | Migrate fragments      | Update all `{{Key}}` → `{{.Key}}` in `workflow-fragments/fragments/*.md`; add conditional example|
| 0004  | Tests + build + ship   | Add/update tests; `go build`; `go install`; `npm run build`; ship hive-deck-pro                 |
