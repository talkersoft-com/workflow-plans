# Real YAML editor — Epic plan 7 (ansible-p7-editor)

## What this workflow does
Replaces the textarea editor with a CodeMirror 6 component: YAML-structure + Ansible-keyword
highlighting, embedded Jinja2 (`{{ }}` parsed as template language), fragment tolerance for
out-of-context chunks, inline diagnostics (client YAML parse + optional server lint findings),
and the inert decoration/hover extension points P8 plugs into. Drop-in: existing consumer props
keep exact semantics.

## Read before starting
- `deck.md`, `PLAN.md` — C-ED contract and the extension-stack design
- `../../master-plans/ANSIBLE-EPIC-CONTRACTS.md` — §9 C-ED is binding verbatim (P8 consumes it)
- `cloud-manager-web/src/components/RoleFileEditor/RoleFileEditor.tsx` and both consumer pages —
  the API being preserved
- `../../master-plans/ANSIBLE-EPIC.md` — epic §7

## Constraints
- **C-ED is binding**: prop names/types per contracts §9; `decorations`/`hoverProvider` ship
  inert this plan — P8 must plug in without editing this plan's files.
- Drop-in compatibility: `initialContent`/`onSave`/`onDirtyChange`, dirty tracking, Cmd/Ctrl-S,
  beforeunload prompt, change-summary flow all byte-identical in behavior.
- Editor buffer is source of truth: save sends buffer text through existing endpoints; the
  component never re-serializes YAML.
- CM6 stays behind the existing lazy boundary; bundle-size guard in CI.
- Theme consumes the existing SCSS custom properties — no second color system.
- Code changes in cloud-manager-web ONLY.
