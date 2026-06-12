# Plan: Semantic decorations — secret/required badges, hover provenance, masking (ansible-p8-decorations)

> Plan 8 of 9 in the Ansible Epic series (windward-steamboat). Conforms to
> `master-plans/ANSIBLE-EPIC-CONTRACTS.md`. Workflow: `feature@cloud-manager`.
> Satisfies epic §8 (semantic decorations from DB metadata, meaningful-not-cosmetic
> highlighting, hover provenance overlay, masking everywhere). Exposes contract **C-DECO**.
> **Depends on: P4, P5, P7** — consumes **C-VAR** (typing metadata), **C-RES** (provenance),
> and **C-ED** (decoration/hover extension points). The high-value differentiator of the epic.

## Objective

Make highlighting *mean something*: `{{ api_token }}` renders with a secret badge and lock
icon because the registry says it's a secret, `{{ db_password }}` shows a required-and-missing
gutter marker because resolution says so, and hovering any variable shows its effective value
(secrets redacted) plus the provenance chain — winner and what it overrode — for the operation
context on screen. Beyond the editor, masking and secret indicators become systematic across
previews, vars editors, logs, and run history.

## Background

- What this plan consumes (all shipped by its dependencies, used as-is):
  - **C-VAR** (contracts §7): `vdef` records carry `kind` (required/default/internal) and
    `sensitivity` (plain/secret) — the classification driving decoration kinds.
  - **C-RES** (contracts §8): `POST api/v1/resolution/preview` returns per-variable state,
    masked `effectiveValue`, winner + overridden chain — the hover payload, verbatim.
  - **C-ED** (contracts §9): `AnsibleYamlEditor` already accepts `decorations` /
    `hoverProvider` and renders them; this plan supplies data, it does not modify the editor.
- Server-side masking already exists at the engine level (P5) and porch redaction
  (`hvs.*` scrub); this plan is the *presentation* layer of the no-plaintext-secrets invariant
  — client surfaces never receive cleartext to begin with, and now they also *signal* secrecy.

## Design

### Decoration pipeline (cloud-manager-web)

New `useVarDecorations(documentContext)` hook:
1. Fetch C-VAR definitions for the document's owner closure (playbook/role → its `vdef` scopes)
   — one batched call, cached per revision.
2. Scan the editor's parse tree for Jinja2 identifier nodes (P7's grammar exposes them) and
   match against definitions by name.
3. Emit `EditorDecoration[]` (C-ED shape): `kind: "secret"` (lock badge + distinct color),
   `"required"` (badge; bold gutter dot when resolution reports `missingRequired`),
   `"vaultRef"` (vault icon on `$secretRef` values and `vault_*` lookups).

Unregistered variables get no decoration — visual silence is the signal to register (links into
P4's harvest flow in P9's UX; here, a quiet tooltip hint only).

**Before** (P7 alone — syntactic only):
```
{{ api_token }}   ← template-expression coloring, same as {{ hostname }}
```
**After** (C-DECO — classification-driven):
```
{{ 🔒 api_token }}      ← secret: lock badge, secret color token
{{ ⚠ db_password }}     ← required + missing in current context: gutter marker + badge
{{ hostname }}           ← plain: ordinary template coloring
```

### Hover provenance (the differentiator, epic §8)

`hoverProvider` implementation: on hover over a matched identifier, call C-RES preview for the
page's operation context (debounced, per-context response cached), render winner
(`host_vars → hvar_… level 14`), the overridden chain, and `effectiveValue` — which for secrets
is the engine's literal `"***"` plus the `sref_…` provenance (cleartext never reaches the
client; contracts §8 invariant). Context selection: editor pages gain an optional context bar
(inventory + host pickers) — without a chosen context, hover shows classification + definition
info only, no resolution call.

### Masking everywhere (epic §8/§10)

Systematic sweep, building on server guarantees:
- **Vars editors** (assignment/blueprint `VarsEditor`, P1 host/group var forms): values for
  secret-classified names render as `***` with a lock chip and a "set via vault ref" affordance
  (writes the `$secretRef` form, never a value — P4's write rules surfaced in UI).
- **Previews and detail pages**: any C-VAR-classified value rendered anywhere shows masked +
  badge (shared `<SecretValue>` component; lock icon per epic §9 secret indicators).
- **Logs/run history**: porch redaction + engine masking already strip values server-side;
  client adds the visual layer — published-secret *paths* in `RunDetailPage` get lock chips,
  and the run vars panel renders from `rsnap` (masked by construction) instead of raw
  `varsSnapshot`.
- Audit: a greppable rule — no component renders a var value except through `<SecretValue>`.

### C-DECO (what P9 consumes)

`useVarDecorations`, `<SecretValue>`, and the context-bar component are exported shared pieces;
P9's composer/preview reuse them rather than re-deriving classification.

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000 | Setup | hv_status + hv_next; record branch in deck.md |
| 0001 | Metadata pipeline | C-VAR batched fetch + cache, identifier scan over P7 tree, decoration emission |
| 0002 | Editor decorations | secret/required/vaultRef badges + gutter markers via C-ED props; theme tokens for both themes |
| 0003 | Hover provenance | Context bar, debounced C-RES calls, provenance tooltip (winner/overridden/masked value) |
| 0004 | Masking sweep | `<SecretValue>`, vars-editor ref-only affordance, run pages on `rsnap`, lock indicators |
| 0005 | Regression + polish | No-context and unregistered-var behavior; perf (hover cache); editor pages without P4/P5 data degrade gracefully |

## Open questions

1. **Context bar persistence**: remember last inventory/host context per playbook
   (localStorage) or always start empty? Default: remember per playbook.
2. **Required-but-missing visibility without a context**: show the required badge always, but
   the "missing" state only with a context selected? Default: yes — missingness is contextual
   by definition (epic §5).
3. **Decoration refresh cadence**: re-scan on every doc change (debounced 300ms) or on save?
   Default: debounced live re-scan; matches lint behavior.
