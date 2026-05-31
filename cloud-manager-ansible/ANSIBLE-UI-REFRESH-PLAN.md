# ANSIBLE-UI-REFRESH-PLAN — cloud-manager-web

## Problem

The ansible UI pages were implemented without following the existing design
system or component library. Compared to the VM pages they live alongside,
the ansible section looks and behaves like a different application:

- Table headers have no styling — no uppercase, no letter-spacing, no muted
  secondary color. Columns are visually indistinguishable from data rows.
- No row hover states anywhere.
- Edit buttons are bare `✎` Unicode characters, not `Button` component instances.
- Modal forms use raw `<label><input>` pairs — no TextInput wrapper, no
  consistent label sizing or gap.
- The Run page uses a native browser `<select>` with `style={{ padding: 8 }}`
  instead of the styled Dropdown component.
- The Run button is `<button style={{ padding: "8px 16px" }}>` — not a Button
  component, so it lacks the variant system, loading state, and disabled opacity.
- There are no status badge components on the history or run-detail pages —
  run statuses are rendered as plain text.
- Missing-collection warnings are a single red `<div>` with no icon, no
  structured layout, and no action link that matches the design language.
- The two-panel layout of "form + VM picker" in AnsibleRunPage is a single
  vertical column — fine on mobile, hard to scan on a wide screen.

The storybook now contains full-fidelity designs for all seven ansible pages.
This plan integrates those designs into the actual app, page by page, without
touching the API layer or adding new endpoints.

## Scope

**In scope — UI refresh only, against the current API:**
- PlaybooksPage
- AnsibleRunPage
- AnsibleHistoryPage
- AnsibleCollectionsPage
- AnsibleRolesPage
- AnsibleRoleDetailPage (file tree + editor)
- AnsibleRunDetailPage (per-target accordion)

**Out of scope for this plan:**
- New API endpoints or data model changes.
- Monaco editor changes (AnsiblePlaybookEditorPage already works).
- Vault / secrets output display.
- SignalR streaming (already wired; RunDetailPage polling stays as-is).
- Backend feature flag changes.
- Any VM detail page changes.

## Reference

All target designs are in:

```
cloud-manager-web/.storybook/src/stories/Ansible/
  AnsiblePlaybooks.stories.tsx     PlaybooksList · PlaybooksEmpty · PlaybooksCreateModal
  AnsibleRun.stories.tsx           RunTrigger · RunMissingCollections · RunSubmitting
  AnsibleHistory.stories.tsx       RunHistory · RunHistoryEmpty
  AnsibleCollections.stories.tsx   CollectionsList · CollectionsEmpty · CollectionsAddModal
  AnsibleRoles.stories.tsx         RolesList · RolesEmpty · RoleDetail
  AnsibleRunDetail.stories.tsx     RunSucceeded · RunFailed · RunRunning
```

The storybook is the spec. If a story shows a behaviour, the page should
match it exactly. If a story state doesn't exist (e.g. a loading skeleton),
the existing Spinner component covers it — no new loading patterns needed.

## Shared primitives to extract first

Before touching individual pages, extract the inline helpers from the story
files into shared SCSS utilities and a new `AnsibleTable` component so each
page doesn't duplicate the same 30 lines of table-header styling:

### `ansible-table.scss` (new, in `src/components/AnsibleLayout/`)

```scss
.ansible-table {
  width: 100%;
  border-collapse: collapse;
  background: var(--color-surface);
  border-radius: 10px;
  border: 1px solid var(--color-border);
  overflow: hidden;

  &__head th {
    padding: 10px 16px;
    text-align: left;
    font-size: 11px;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.6px;
    color: var(--color-text-muted);
    border-bottom: 1px solid var(--color-border);
    background: var(--color-surface-raised);
  }

  &__body tr {
    border-bottom: 1px solid var(--color-border);
    transition: background 0.1s;
    cursor: pointer;

    &:hover {
      background: var(--color-surface-raised);
    }

    td {
      padding: 14px 16px;
      font-size: 14px;
    }
  }
}
```

### `StatusBadge` component (new, in `src/components/StatusBadge/`)

```tsx
// Covers all run statuses: Succeeded · Failed · Running · Queued · Cancelled
// Running state gets CSS pulse animation on the dot.
// Used by: AnsibleHistoryPage, AnsibleRunDetailPage, PlaybooksPage (last-run column)
<StatusBadge status="Succeeded" />
<StatusBadge status="Running" />   // dot pulses
```

Color map (matches storybook):
- Succeeded → `#0e8420`
- Failed → `#df382c`
- Running → `#335280` (pulse animation)
- Queued → `var(--color-text-muted)`
- Cancelled → `#f99b11`

### `InstallBadge` component (new, or export from StatusBadge)

Collection install statuses: Installed · Installing · Failed · Removed · Pending.
Rendered as a colored pill (background + text), not a dot.

## Page-by-page changes

### 1. PlaybooksPage

**File:** `src/pages/PlaybooksPage.tsx`

Changes:
- Wrap table in `ansible-table` class.
- Replace raw `<th style={{ padding: "8px" }}>` with `ansible-table__head`.
- Replace `<button onClick={...}>✎</button>` with `<Button variant="ghost" size="small">Edit</Button>`.
- Replace `<button style={{ color: "var(--color-error)" }}>Delete</button>` with `<Button variant="danger" size="small">`.
- Replace `<button className="page__action-btn">` with `<Button>` (already the correct class, but should use the component for consistency).
- Add `LastRunStatus` column using `StatusBadge`.
- Add row hover state via `ansible-table__body tr:hover`.
- Modal form: replace bare `<label><input>` pairs with `TextInput` component (already exists).
- Modal action buttons: replace bare `<button>` with `Button` component instances.

### 2. AnsibleRunPage

**File:** `src/pages/AnsibleRunPage.tsx`

Changes:
- Switch from single-column layout to `display: grid; grid-template-columns: 1fr 340px` — form on left, VM picker card on right (matches `RunTrigger` story).
- VM picker: extract the current `VmMultiSelect` into a card with a header showing selected count. Card gets `border: 1px solid var(--color-border); border-radius: 10px; overflow: hidden` styling.
- Playbook `<select>`: keep native select but apply the styled wrapper from the story (custom arrow, proper border-radius, accent-color on focus).
- Vars override `<textarea>`: add `border-radius: 8px; border: 1px solid var(--color-border)` — currently unstyled.
- Missing-collections section: replace the red `<div style={{ color: "var(--color-error)" }}>` with the warning banner from `RunMissingCollections` story — amber background, warning icon SVG, collection name pills, action link.
- Run button: replace `<button style={{ padding: "8px 16px" }}>` with `<Button>` component.
- Field labels: replace `<div style={{ fontWeight: 600 }}>` wrappers with the `Field` pattern from story (13px, 600 weight, secondary color label above the input).

### 3. AnsibleHistoryPage

**File:** `src/pages/AnsibleHistoryPage.tsx`

The page structure is already reasonable (uses `RunHistoryGrid`). Changes are
in the `RunHistoryGrid` component itself:

**File:** `src/components/RunHistoryGrid/RunHistoryGrid.tsx`

Changes:
- Apply `ansible-table` class.
- Replace the current status rendering (text only, or color-coded text) with
  `<StatusBadge status={item.status} />`.
- Table headers: uppercase, letter-spacing, muted color (currently unstyled).
- Row hover state via `ansible-table__body tr:hover`.
- Add duration column (already in story data; calculate from `startedAt` /
  `completedAt` if both present, otherwise `—`).
- "View →" link per row in rightmost column.

**File:** `src/pages/AnsibleHistoryPage.tsx`

- Add VM filter `<select>` in a filter bar above the table (matches story
  filter row). Currently `VmFilterCombobox` is present but unstyled.
- Show total run count to the right of the filter bar.

### 4. AnsibleCollectionsPage

**File:** `src/pages/AnsibleCollectionsPage.tsx`

Changes:
- Apply `ansible-table` class.
- Table headers: Collection, Description, one column per controller host, actions.
- Replace per-host install status text with `<InstallBadge status={...} />`.
- Action buttons per row: "Install" (secondary), "Reinstall" (ghost), "Remove"
  (danger) — all using `Button` component with `size="small"`.
- Modal: replace bare inputs with `TextInput`. Add namespace/name preview line
  (matches `CollectionsAddModal` story).
- Empty state: use `EmptyState` component (already exists).

### 5. AnsibleRolesPage

**File:** `src/pages/AnsibleRolesPage.tsx`

Changes:
- Apply `ansible-table` class.
- Add "Used by" column showing playbook attachment count as an accent-colored
  pill (e.g. "3 playbooks").
- Add "Files" column showing file count.
- Action buttons: "Edit" (ghost), "Delete" (danger) using `Button size="small"`.
- Modal form: `TextInput` for name and description.

### 6. AnsibleRoleDetailPage

**File:** `src/pages/AnsibleRoleDetailPage.tsx` and `src/components/RoleFileTree/`

Changes:
- Apply the two-panel layout from the `RoleDetail` story: file tree left
  (220px), editor right (flex 1), both with `border-radius: 10px` cards.
- File tree: collapsible directory sections with animated chevron.
- Editor pane header: file path (monospace) + "Save" button right-aligned.
- Breadcrumb: "← Roles / {role-name}" with role-name in monospace.
- Subtitle: file count + playbook attachment count below the title.

### 7. AnsibleRunDetailPage

**File:** `src/pages/RunDetailPage.tsx` (currently at `/ansible/runs/:runId`)

Changes:
- Stats bar: four cards in a `repeat(4, 1fr)` grid — Started, Completed,
  Targets, Playbook. Currently these are inline text, not card components.
- Per-target rows: replace the current list with accordion cards matching
  the `RunDetail` story — dot indicator, vm name, IP address, ok/changed/failed
  counts inline, click to expand output `<pre>`.
- Output `<pre>`: `background: var(--color-surface-raised); border-top: 1px
  solid var(--color-border); font-size: 12px; line-height: 1.6`.
- Overall status badge: `<RunStatusBadge>` (large variant, pill-shaped) in
  the page header next to the playbook name.
- "Cancel run" button: visible only when status is `Queued` or `Running`,
  using `<Button variant="danger">`.

## What we are NOT changing

- `AnsiblePlaybookEditorPage` — Monaco editor works; leave it alone.
- `AnsibleSubNav` — already correct.
- `AnsibleLayout` — already correct.
- Any Redux slice logic — data fetching is fine; only rendering changes.
- Any API service methods — no changes needed.
- Route structure — no changes.
- `FeatureGate` wrapping — stays exactly as-is.

## Phases

Each phase is a single PR to `cloud-manager-mcp` (the web app repo).

| Phase | Scope | Storybook reference |
|-------|-------|---------------------|
| 1 | `ansible-table.scss` + `StatusBadge` + `InstallBadge` shared primitives | All stories |
| 2 | PlaybooksPage refresh | PlaybooksList · PlaybooksCreateModal |
| 3 | AnsibleRunPage refresh | RunTrigger · RunMissingCollections |
| 4 | RunHistoryGrid + AnsibleHistoryPage refresh | RunHistory |
| 5 | AnsibleCollectionsPage refresh | CollectionsList · CollectionsAddModal |
| 6 | AnsibleRolesPage refresh | RolesList |
| 7 | AnsibleRoleDetailPage two-panel layout | RoleDetail |
| 8 | AnsibleRunDetailPage stats bar + accordion | RunSucceeded · RunFailed · RunRunning |

Phases 1–4 are the highest impact (most-used pages). Phases 5–8 can follow in
any order once 1–4 are merged.

## Definition of done

- Each page visually matches its storybook story to the eye.
- No inline `style={{ padding: "8px" }}` or similar one-off overrides remain
  on any ansible page.
- All interactive elements (buttons, links, edit actions) use the `Button`
  component or an established CSS class — no bare `<button>` with inline style.
- `ansible-table` class is used on every list table in the ansible section.
- `StatusBadge` is used everywhere a run status is displayed.
- Storybook stories remain runnable and match the live pages.
- No regressions on VM pages, Settings, or Dashboard.
