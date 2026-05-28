# WEB-PLAN — cloud-manager-web ansible UI

## Problem

The existing ansible UI is sloppy: ansible controls are embedded inside VM detail pages (`VmPlaybooks` component, ansible-related sections on `VmDetailPage`), mixing two unrelated concerns. A VM has no business showing ansible content. Beyond the architectural mess, the UI also lacks a YAML editor, role-library management, a dedicated run-trigger workflow, a proper history view with filtering, and any in-UI control over the `playbooks` feature flag (the operator has to flip it with `curl` today). This plan redesigns the ansible surface from scratch as a clean, self-contained section of the app, plus an always-visible `SettingsPage` for feature-flag management.

## Architectural decision — clean separation

State this explicitly so no follow-on change accidentally re-mixes:

- Ansible is managed from the **Ansible section only**, at `/ansible/*`.
- **No ansible UI on VM pages.** The `VmPlaybooks` component is **deleted**. Any ansible-related panels, tabs, buttons, or links on `VmDetailPage` are **removed**.
- A VM ↔ playbook assignment is created from the Ansible side (open a playbook, pick VMs to attach or run against). It is NOT created from the VM side.
- `PlaybooksPage` and `PlaybookDetailPage` (existing) are **relocated/replaced** under the `/ansible` prefix.
- Old `/playbooks/*` and `/runs/*` routes get one release cycle of redirects to `/ansible/playbooks/*` and `/ansible/runs/*`, then are deleted.
- To answer "which VMs is this playbook on?", the operator uses the `AttachedVmsPanel` inside the playbook detail page — not the VM page.

## Code editor decision

Recommend **Monaco**. Justification (3–5 lines):

- Best-in-class YAML/JSON support out of the box, including syntax highlighting, validation, and bracket matching.
- Built-in diff view — critical for `RevisionHistoryDrawer`'s "preview historic revision" UX. CodeMirror 6 has no built-in diff; we'd build it ourselves.
- Familiar UX (VS Code) for the operator audience; minimal learning curve.
- Bundle-size cost (~2 MB gzipped) is real but easily mitigated by **dynamic import on editor routes only** (`React.lazy(() => import('./components/RoleFileEditor'))`). The list/detail pages stay light; only `/ansible/.../edit` and `/ansible/roles/:id` pull Monaco. Vite handles the code split automatically.

CodeMirror 6 was considered as the lighter alternative but rejected on the diff-view requirement. A `textarea` + Prism approach was rejected as too weak an editing experience for multi-file ansible role authoring.

## Nav structure

Top-level entries in the app shell:

- Dashboard (always visible)
- Hosts (always visible)
- VMs (always visible)
- **Ansible** (flag-gated on `feature_flags.playbooks.enabled`)
- **Settings** (always visible)

The **Ansible** top-level expands to a sub-nav inside the Ansible section:

- Playbooks → `/ansible/playbooks`
- Roles → `/ansible/roles`
- Run → `/ansible/run`
- History → `/ansible/history`

When `feature_flags.playbooks.enabled === false`, the entire Ansible entry disappears from the nav AND `/ansible/*` routes return 404 via `FeatureGate`. When the flag is on but `healthy === false`, the Ansible entry is visible but each sub-page shows a banner pointing at `ANSIBLE-INSTALL-PLAN.md`.

**Settings** is always visible (never flag-gated) because that's where the operator turns the `playbooks` flag on in the first place.

## New routes

All routes nested under `/ansible` are gated via `FeatureGate` with the `playbooks` flag. `/settings` is NOT gated.

| Route | Page | Purpose |
|-------|------|---------|
| `/ansible` | redirect → `/ansible/playbooks` | Entry point |
| `/ansible/playbooks` | `AnsiblePlaybooksPage` | List all playbooks; create new |
| `/ansible/playbooks/:playbookId` | `AnsiblePlaybookDetailPage` | Read-only overview: revision history, `AttachedRolesPanel`, `AttachedVmsPanel`, "Edit YAML" + "Run on VMs" buttons |
| `/ansible/playbooks/:playbookId/edit` | `AnsiblePlaybookEditorPage` | Monaco editor + `RevisionHistoryDrawer` + `AttachedRolesPanel` + `AttachedVmsPanel` |
| `/ansible/playbooks/:playbookId/roles/:roleId` | `AnsiblePlaybookLocalRolePage` | File tree + editor for a playbook-local role |
| `/ansible/roles` | `AnsibleRolesPage` | Global role library — list, create, delete (with 409 surface if attached) |
| `/ansible/roles/:roleId` | `AnsibleRoleDetailPage` | `RoleFileTree` + `RoleFileEditor` + `RevisionHistoryDrawer` |
| `/ansible/run` | `AnsibleRunPage` | Pick playbook + target VMs + var overrides; POSTs to multi-VM run endpoint |
| `/ansible/history` | `AnsibleHistoryPage` | Paginated table of all runs; filter by playbook or by VM |
| `/ansible/history/:runId` | `AnsibleRunDetailPage` | Per-target status, live polling on `Running`, `RunCancelButton`, secrets list |
| `/settings` | `SettingsPage` | Always visible — `FeatureFlagToggle` for every flag |
| `/playbooks/*`, `/runs/*` | redirects | One release cycle, then delete |

## New pages

- **`AnsiblePlaybooksPage`** — replaces `PlaybooksPage`. Searchable list with "Create" button. Each row → `AnsiblePlaybookDetailPage`.
- **`AnsiblePlaybookDetailPage`** — replaces `PlaybookDetailPage`. Read-only summary: name, description, argument schema (parsed for display), revision timeline, `AttachedRolesPanel`, `AttachedVmsPanel`. Action buttons: "Edit YAML" → editor route, "Run on VMs" → `AnsibleRunPage` pre-populated with this playbook.
- **`AnsiblePlaybookEditorPage`** — full Monaco editor for the playbook YAML. Save flow as specified below. Hosts `RevisionHistoryDrawer`, `AttachedRolesPanel`, and `AttachedVmsPanel` as slide-outs or sidebars.
- **`AnsiblePlaybookLocalRolePage`** — same shape as `AnsibleRoleDetailPage` but scoped to a playbook-local role. File tree on left, editor on right, revision drawer.
- **`AnsibleRolesPage`** — global role library. List + search + create. Delete button shows 409 toast with "detach from N playbooks first" message.
- **`AnsibleRoleDetailPage`** — `RoleFileTree` (left), `RoleFileEditor` (right), `RevisionHistoryDrawer` (slide-out). Tree built client-side from the flat `GET .../file` response.
- **`AnsibleRunPage`** — two-step form:
  1. Pick a playbook from a searchable dropdown (auto-loads if arrived via "Run on VMs" link).
  2. Pick one or more VMs via `VmMultiSelect`, with optional per-VM var overrides expanded inline.
  "Run" button calls `POST /api/v1/playbook/{id}/run` with `{ vmIds, varsOverride }`. On success, navigate to `AnsibleRunDetailPage` for the new run.
- **`AnsibleHistoryPage`** — paginated table of every `PlaybookRun`. Filter modes (via tabs or dropdown — `RunHistoryFilters`):
  - **By playbook** — group/filter by playbook name
  - **By VM** — group/filter by target VM
  Columns: run status, started_at, duration, target count, playbook. Row click → `AnsibleRunDetailPage`.
- **`AnsibleRunDetailPage`** — replaces `RunDetailPage` for ansible runs. Shows overall run status, `RunTargetsList` (per-target rows), secrets list (using existing reveal-on-click). When status is `Running`, polls every N seconds (default 2s) to refresh status and stream output; polling stops on terminal state. Includes `RunCancelButton` (visible only when status is `Queued` or `Running`).
- **`SettingsPage`** — always visible at `/settings`. Hosts `FeatureFlagToggle` for every entry returned by `GET /api/v1/feature-flags` (today: just `playbooks`; designed to scale as more flags land). Shows the current `enabled` state, `healthy` indicator, and `last_probed_at` per flag. Toggling calls `PATCH /api/v1/admin/feature-flags/{key}` and refetches the flags list on success. This is the operator's in-UI control to enable or disable the ansible section without using `curl`.

## New components

- `RoleFileTree` — collapsible file-path navigator. Props: `files[]`, `selectedPath`, `onSelect`. Builds a tree client-side from the flat file list. Shared between `AnsibleRoleDetailPage` and `AnsiblePlaybookLocalRolePage`.
- `RoleFileEditor` — wraps Monaco. Props: `path`, `content`, `dirty`, `onSave`. PATCH on save; success invalidates the revision list slice.
- `RevisionHistoryDrawer` — slide-out panel listing revisions newest-first. Click a revision to preview its content side-by-side (Monaco diff view); "Restore" button calls `POST .../restore`. Used on role-file pages and the playbook editor.
- `AttachedRolesPanel` — used inside `AnsiblePlaybookEditorPage` and `AnsiblePlaybookDetailPage`. Two tabs:
  - **Global roles** — list of `PlaybookGlobalRoleRef`. Attach button opens a picker over the global library. Detach calls DELETE.
  - **Local roles** — list of `PlaybookRole`. "Open editor" link → `/ansible/playbooks/:pid/roles/:rid`. Create / delete inline.
- `AttachedVmsPanel` — reverse-direction view used on `AnsiblePlaybookDetailPage` and `AnsiblePlaybookEditorPage`. Lists every VM the playbook is attached to via `VmPlaybookAssignment` (vm name, last run status, last applied at, link to per-VM detail page). This is the from-the-playbook-side view of the many-to-many; the VM-side view does NOT exist anymore (deleted per the clean-separation decision). This panel is how an operator answers "which VMs is this playbook deployed on?" without leaving the Ansible section.
- `RunTargetsList` — table of `PlaybookRunTarget` rows with status badge, stats summary, collapsible per-target output. Supports live refresh when the parent run is active.
- `VmMultiSelect` — searchable multi-select of VMs for the run trigger form. Shows VM name + host. Supports per-VM var override expansion.
- `RunHistoryFilters` — tab/dropdown switcher for "by playbook" vs "by VM" filtering on `AnsibleHistoryPage`.
- `RunCancelButton` — on `AnsibleRunDetailPage`. Visible only when run status is `Queued` or `Running`. Calls `POST /api/v1/run/{id}/cancel` and triggers a re-fetch. Disabled with a tooltip when the run is already terminal.
- `FeatureFlagToggle` — used inside `SettingsPage`. Renders a labeled switch per flag, plus the `healthy` and `last_probed_at` indicators next to each flag so the operator can see "enabled but broken" states distinctly. Toggle calls `PATCH /api/v1/admin/feature-flags/{key}`.

## New Redux slices

- `ansiblePlaybooksSlice` — list / get / create / update / delete. Update PATCH invalidates the matching revision list (since the server creates a new revision on content change).
- `ansibleRolesSlice` — global role library; caches file tree summaries and per-file content keyed by `(roleId, fileId)`.
- `roleFileRevisionsSlice` — keyed by `(roleId, fileId)`; revision lists + historic content for the drawer.
- `playbookRevisionsSlice` — keyed by `playbookId`; revision lists + historic content.
- `playbookLocalRolesSlice` — keyed by `playbookId`; mirrors the global structure but scoped per playbook.
- `runTargetsSlice` — keyed by `runId`. Supports polling updates; polling state is per-run.
- `ansibleRunsSlice` — paginated run list for the history page; filter params live in slice state so deep-links to filtered views work.
- `featureFlagsSlice` — fetches `GET /api/v1/feature-flags` on app load and on a polling interval (60s default) so the UI tracks `healthy` state. Powers both `FeatureGate` checks across the Ansible section and `FeatureFlagToggle` on `SettingsPage`. If a slice already exists for `FeatureGate`, extend it rather than duplicating.

## API client (`services/api.ts`) extensions

One typed method per endpoint defined in `API-PLAN.md`. All new endpoints use the same `fetch` helpers already in use. New TypeScript interfaces in `src/types/ansible.ts` covering: `AnsibleRole`, `RoleFile`, `RoleFileSummary`, `RoleFileRevision`, `PlaybookRevision`, `PlaybookGlobalRoleRef`, `PlaybookLocalRole`, `PlaybookLocalRoleFile`, `PlaybookLocalRoleFileRevision`, `PlaybookRunTarget`, `PlaybookRun`, `MaterializedPlaybook` (for type-sharing with vorch's API client if ever needed), and the request bodies for the new POST endpoints.

## Deletions — existing code to remove

- `src/components/VmPlaybooks/` — **delete entirely**.
- Any ansible-related panels, tabs, sections, or buttons on `src/pages/VmDetailPage.tsx` — **remove**.
- `src/pages/PlaybooksPage.tsx` and `src/pages/PlaybookDetailPage.tsx` — **replace** with the new `/ansible/playbooks` equivalents. Keep the old route paths as redirects for one release cycle, then delete the old files entirely.
- `src/pages/RunDetailPage.tsx` — same: replaced by `AnsibleRunDetailPage` at `/ansible/runs/:runId` with a one-release redirect.
- `src/store/slices/assignmentsSlice.ts` — audit. If `AttachedVmsPanel` and the new ansible slices subsume its data, delete it. If anything still consumes it, narrow it to the minimum and document why.

## Save UX

Applies to `RoleFileEditor` and the playbook YAML editor identically.

- Editor is dirty when content differs from the loaded version.
- "Save" button disabled when not dirty.
- Cmd/Ctrl-S triggers save.
- On save: optimistic editor state update → PATCH network call → on success invalidate and re-fetch the revision list (the new revision should appear in the drawer) → reset dirty flag. On failure: toast the error message, keep the dirty flag set so retry is possible.
- Browser-leave prompt when dirty (`window.beforeunload` handler installed while dirty).
- React-Router navigation prompt when dirty (warn before changing routes).

## Phases

Ordered smallest-shippable-slice first. Each phase is independently mergeable and testable.

1. **`SettingsPage` + `FeatureFlagToggle`** (the operator can flip the `playbooks` flag from the UI before anything else lights up — this is the bootstrap).
2. **Ansible nav section + empty sub-pages scaffold** — routes wired under `/ansible/*`, gating in place, sub-nav rendered. Pages render "Coming soon."
3. **Remove `VmPlaybooks` from `VmDetailPage`; redirect old `/playbooks` and `/runs` routes** to their `/ansible/...` equivalents.
4. **`AnsiblePlaybooksPage` + `AnsiblePlaybookDetailPage`** — read-only, no editor. Pulls from existing `playbooksSlice` (extended to `ansiblePlaybooksSlice`).
5. **`AnsibleRolesPage` + `AnsibleRoleDetailPage`** — read-only file tree, no editor. New `ansibleRolesSlice`.
6. **Monaco wiring** behind dynamic import. `RoleFileEditor` save flow + `RevisionHistoryDrawer` with diff view.
7. **Playbook YAML editor** (`AnsiblePlaybookEditorPage`) — same editor component, plus `AttachedRolesPanel` and revision drawer.
8. **`AttachedVmsPanel`** on detail + editor pages — the reverse-direction view.
9. **`AnsiblePlaybookLocalRolePage`** — reuses phases 6 and 7's components, scoped to a playbook-local role.
10. **`AnsibleRunPage`** — trigger form with `VmMultiSelect`. Wires to the new `POST /api/v1/playbook/{id}/run` endpoint.
11. **`AnsibleHistoryPage` + `AnsibleRunDetailPage`** — paginated table with `RunHistoryFilters`; per-run detail with `RunTargetsList`.
12. **Live polling on active runs** in `AnsibleRunDetailPage`, plus `RunCancelButton` wired to the new `POST /api/v1/run/{id}/cancel` endpoint.
