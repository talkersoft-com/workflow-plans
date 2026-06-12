# Plan: Real YAML editor — CodeMirror 6, embedded Jinja2, fragments, inline lint (ansible-p7-editor)

> Plan 7 of 9 in the Ansible Epic series (windward-steamboat). Conforms to
> `master-plans/ANSIBLE-EPIC-CONTRACTS.md`. Workflow: `feature@cloud-manager`.
> Satisfies epic §7 (real editor component, YAML-structure highlighting, Ansible semantics,
> embedded Jinja2, fragment tolerance, inline lint surfacing, round-trip integrity at the
> editor). Exposes contract **C-ED** (contracts §9) — consumed by P8. Depends on: none
> (web-only foundation; P6's lint findings and P3's fidelity layer light up when present).

## Objective

Replace the plain `<textarea>` editor with a real CodeMirror 6 component: YAML-structure
highlighting (keys/values, anchors/aliases, block scalars, comments, indent guides), Ansible
keywords and module names highlighted distinctly, `{{ … }}` Jinja2 expressions parsed as
embedded template language rather than plain strings, tolerance for fragment chunks that aren't
valid standalone documents, and a diagnostics surface (squiggles + gutter) fed by client-side
YAML parsing and — where available — server lint findings. Ships as a drop-in replacement:
every current consumer keeps its exact props.

## Background

- Today (`cloud-manager-web`): `RoleFileEditor.tsx` is a `<textarea>` with monospace CSS, dirty
  tracking, Cmd/Ctrl-S, change-summary field; lazy-loaded via `React.lazy` in
  `AnsiblePlaybookEditorPage` and `AnsibleRoleDetailPage`. The code already documents an
  upgrade path "without changing consumer API" — this plan is that upgrade, with CM6 chosen
  over Monaco (epic §7 names CM6 lighter/more embeddable; bundle size matters in a Vite SPA,
  and CM6's extension/decoration model is exactly what P8 needs).
- Stack realities: React 18 + Vite + SCSS design tokens with dark/light themes via
  `[data-theme]` CSS variables — the editor theme must consume the same tokens.
- The editor buffer is the source of truth (epic §7): save sends the buffer text through the
  existing save endpoints; nothing in this plan re-serializes YAML. (Byte-level write fidelity
  into revisions is P3's server-side guarantee.)

## Design

### Component — C-ED verbatim (contracts §9)

New `AnsibleYamlEditor` (replaces `RoleFileEditor` internals; the file keeps its lazy-loaded
module boundary):

```tsx
<AnsibleYamlEditor
  initialContent={string}
  onSave={(content, changeSummary?) => Promise<void>}
  onDirtyChange={(dirty) => void}
  fragmentMode={boolean}
  decorations={EditorDecoration[]}        // P8 supplies; inert (empty) this plan
  diagnostics={EditorDiagnostic[]}
  hoverProvider={(varName, pos) => Promise<HoverContent | null>}  // P8 wires later
/>

type EditorDecoration = { from: number; to: number; kind: "secret" | "required" | "vaultRef"; varName: string }
type EditorDiagnostic  = { from: number; to: number; severity: "error" | "warning" | "info"; message: string; rule?: string }
```

Existing props (`initialContent`, `onSave`, `onDirtyChange`) keep their exact semantics —
consumers compile unchanged. New props are optional with inert defaults.

**Before** (today — `RoleFileEditor.tsx`):
```tsx
<textarea style={{ fontFamily: "monospace" }} value={content} onChange={…} />
```
**After** (C-ED): the component above — CM6 `EditorView` with the extension stack below;
`{{ api_token }}` is a parsed template node carrying a decoration target, not string content.

### CM6 extension stack

- **Language**: `@codemirror/lang-yaml` (Lezer YAML) as the host grammar; Jinja2 embedded in
  scalar values via a small Lezer grammar for `{{ … }}` / `{% … %}` mounted with nested-parser
  support — template delimiters, identifiers, and filters get their own highlight tags.
- **Ansible semantics**: highlight-tag overlay for play/task keywords (`hosts`, `vars`,
  `tasks`, `handlers`, `when`, `notify`, …) and module-name position (first non-meta key of a
  task mapping) — a styling pass over the YAML tree, not a fork of the grammar.
- **Fragment tolerance** (`fragmentMode`): chunks are virtually wrapped before parse (synthetic
  document + re-indent), positions mapped back so highlighting/diagnostics never error on a
  "missing document start"; relative indentation preserved on save (epic §7).
- **Diagnostics**: CM6 `linter()` source merging (a) client-side YAML syntax errors from the
  Lezer tree, (b) caller-supplied `diagnostics` (P6 `plint` findings fetched by the host page
  when the API exposes them — graceful absence otherwise). Squiggles + gutter markers + lint
  panel.
- **Theme**: one CM6 theme reading the existing SCSS custom properties (`--color-*`), correct
  under both `[data-theme]` values; no second color system.
- **Extension points for P8**: a `decorations` → `Decoration.mark/widget` compartment and the
  `hoverProvider` → `hoverTooltip` bridge ship now (inert), so P8 plugs in without touching
  this plan's files again.

### Bundle discipline

CM6 packages (`@codemirror/state`, `view`, `language`, `lint`, `lang-yaml`, custom Jinja2
grammar) stay behind the existing `React.lazy` boundary — list pages pay zero bytes; the editor
chunk loads on demand. Budget asserted in CI via Vite build-size check (±10% guard).

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000 | Setup | hv_status + hv_next; record branch in deck.md |
| 0001 | CM6 foundation | Packages, `AnsibleYamlEditor` with legacy props parity, theme from SCSS tokens, lazy-chunk budget |
| 0002 | YAML + Ansible semantics | lang-yaml integration, keyword/module overlay, indent guides |
| 0003 | Embedded Jinja2 | Lezer template grammar, nested mounting in scalars, highlight tags |
| 0004 | Fragments + diagnostics | fragmentMode wrap/map, client parse lint, `diagnostics` prop merge, gutter/panel |
| 0005 | Extension points + regression | decorations/hoverProvider compartments (inert), both consumer pages swapped, dirty/save/beforeunload behavior byte-identical |

## Open questions

1. **Jinja2 grammar scope**: full statement blocks (`{% if %}`) or expressions-only (`{{ }}`)
   first? Default: expressions + statement *delimiters* highlighted; deep statement parsing
   later if P8/P9 need it.
2. **Existing community grammar vs in-repo Lezer grammar** for Jinja2? Default: small in-repo
   grammar (the npm options are unmaintained; ~150 lines covers our need).
3. **Lint fetch ownership**: host page fetches `plint` findings and passes `diagnostics`, or
   the editor self-fetches? Default: host page — keeps the component API-agnostic (C-ED stays a
   pure component contract).
