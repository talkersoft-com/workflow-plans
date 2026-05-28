# Plan: Integrate the Ansible Playbook YAML editor into the UI

## Objective

Make the existing playbook YAML editor at `/ansible/playbooks/:pid/edit` discoverable and dismissible. Operators on `/ansible/playbooks` (list) and `/ansible/playbooks/:pid` (detail) get an Edit affordance that lands them in the editor; inside the editor, a Close control returns them to the page they came from. End state: an operator can open the Ansible section, navigate to a playbook, click Edit, modify the YAML, save, and Close back to wherever they started — with no URL-typing and no dead-end pages.

## Background

What already works in this repo:

- **Route**: `cloud-manager-web/src/App.tsx:147` declares `<Route path="/ansible/playbooks/:pid/edit" element={<AnsibleLayout><AnsiblePlaybookEditorPage /></AnsibleLayout>} />`. The editor is wrapped in `AnsibleLayout` so the sub-nav tab strip stays visible.
- **Editor page** (`src/pages/AnsiblePlaybookEditorPage.tsx`, 93 lines): wraps `<FeatureGate flag="playbooks">`, fetches the playbook via `fetchPlaybook(pid)`, lazy-loads `@/components/RoleFileEditor/RoleFileEditor` as the YAML editor, calls `updatePlaybook({pid, body:{content, changeSummary}})` on save (which the API auto-rolls into a new `vm.playbook_revisions` row), and renders a Revisions slideover from a "Revisions" button. There are currently no `navigate(...)` calls in this file — no Close / Cancel / Done control.
- **API + DB**: PATCH `/api/v1/ansible/playbooks/{pid}` and revisions infrastructure are already shipped. `vm.playbooks` (`pb_EMEJCH08GG` "pg" is a real row today), `vm.playbook_revisions`, and the per-revision content storage are all live.

What's missing:

1. **Entry points**: `PlaybookDetailPage.tsx` (62 lines) has no Edit button. `PlaybooksPage` list rows have no edit affordance. Operators have no way to reach `/edit` other than typing it.
2. **Close affordance**: editor has no exit button at all.
3. **Referrer memory**: a naive `navigate(-1)` breaks under refresh / bookmark / deep-link cold-load. We need explicit referrer tracking.
4. **Layout drift**: editor uses `height: calc(100vh - 120px)` from when `AnsibleLayout` (48px sub-nav) didn't exist. Adding header buttons makes this worse.

## Design

UI-only change in `cloud-manager-web`. No API, DB, or vorch changes.

### Routing & referrer pattern

We use react-router-dom's `location.state` to carry the referrer forward without polluting the URL.

**Entry point** (`PlaybookDetailPage`, `PlaybooksPage`):
```tsx
const location = useLocation();
const navigate = useNavigate();

const openEditor = () => navigate(
  `/ansible/playbooks/${pid}/edit`,
  { state: { from: { pathname: location.pathname, search: location.search } } }
);
```

**Close** (editor page):
```tsx
const location = useLocation();
const navigate = useNavigate();

const close = () => {
  const from = (location.state as { from?: { pathname: string; search?: string } } | null)?.from;
  if (from?.pathname && from.pathname.startsWith("/ansible/")) {
    navigate(from.pathname + (from.search ?? ""));
  } else {
    navigate(`/ansible/playbooks/${pid}`);
  }
};
```

The `startsWith("/ansible/")` guard is defensive: even if someone constructs a malicious `from` state, Close stays inside the Ansible section. Cold-load (`location.state === null`) hits the detail-page fallback.

### Entry points

Two surfaces, intentionally limited:

| Surface | Affordance | Position |
|---|---|---|
| `PlaybookDetailPage` header | Primary button labeled "Edit" with a pencil icon | Top-right of header, next to the existing actions |
| `PlaybooksPage` list row | Icon-only pencil button | Trailing column of each row, beside the existing row controls |

**Out of scope** for entry points: Run page, History page, role-detail page. We're not adding edit shortcuts there — keep the discoverable surface inside the Playbooks tab.

Both entry points are inside the existing `<FeatureGate flag="playbooks">` boundary by virtue of being on Playbooks-tab pages. No extra gating needed.

### Close affordance in the editor

Editor header gets two controls instead of one:

- Left of header (existing): playbook name + description.
- Right of header (new):
  1. "Revisions" (existing button, kept).
  2. **"Close" button** (new, primary on a brand-light variant or text-only — settle in implementation): pencil-X icon + text. Position: rightmost in the header.

Click handler runs the `close()` function above.

### Unsaved-changes guard — Option A (recommended)

`RoleFileEditor` is the shared YAML editor. It currently does not expose dirty state to the parent page (saving uses an `onSave` callback). We need:

- Add an `onDirtyChange?: (dirty: boolean) => void` prop to `RoleFileEditor`. The editor already tracks internal content vs. its initial prop; expose that via a `useEffect` that compares and calls the callback when the value changes.
- Editor page tracks `const [dirty, setDirty] = useState(false)` and passes `setDirty` as `onDirtyChange`.
- Close handler: if `dirty`, show a confirmation (browser `window.confirm("Discard unsaved changes?")` is acceptable for v1 — a styled modal can come later). On confirm → navigate. On cancel → no-op.
- Also wire a `window.addEventListener("beforeunload", ...)` for the dirty case so the browser warns on tab close / refresh.

Tradeoff: `window.confirm` is ugly but ships in one PR. A custom `ConfirmDialog` component would be cleaner but adds a component we haven't built. Pick `window.confirm` for v1 unless a `ConfirmDialog` already exists in the codebase (Phase 1 verifies this with a grep).

### Layout polish

Replace the magic-number height with a flex layout:

```tsx
// AnsiblePlaybookEditorPage outer wrapper
<div style={{ display: "flex", flexDirection: "column", height: "100%", minHeight: 0 }}>
  <div className="page__header" style={{ padding: 16, flexShrink: 0 }}>...</div>
  <div style={{ flex: 1, minHeight: 0 }}>
    <Suspense fallback={<Spinner />}>
      <LazyEditor ... />
    </Suspense>
  </div>
</div>
```

`AnsibleLayout` already gives its children `flex: 1, minHeight: 0, overflow: auto` (verified at `components/AnsibleLayout/AnsibleLayout.tsx:17-19`), so this flex chain bottoms out into a usable editor height regardless of how many sub-nav rows exist above.

### What stays the same

- `RoleFileEditor.tsx` internals beyond the new `onDirtyChange` prop.
- `updatePlaybook` thunk, `playbooksSlice`, all API endpoints.
- The Ansible sub-nav (`AnsibleSubNav.tsx`) — Playbooks tab already stays highlighted on `/ansible/playbooks/:pid/edit` because of its prefix-matching active-tab logic.
- `<FeatureGate flag="playbooks">` boundary — editor + entry points all sit inside it.

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000  | Setup | hv_status + hv_init/hv_next; confirm working branch |
| 0001  | Editor close + referrer | Add Close button to `AnsiblePlaybookEditorPage` with `from`-aware navigation; wire `onDirtyChange` from `RoleFileEditor`; add `beforeunload` guard; tighten the magic-number height to flex |
| 0002  | Entry points | Add Edit button on `PlaybookDetailPage` header; add pencil-icon edit affordance on each `PlaybooksPage` list row; both pass `location.state.from` |
| 0003  | Build + deploy | `npx tsc --noEmit`; `npm run build`; `sudo rm -rf dist`; `sudo python3 cloud-manager-api/scripts/web/install-web-app.py` |
| 0004  | Verify in browser | Drive both flows via Playwright on `ubuntu-server.talkersoft.com/ansible/playbooks`: open editor from list, edit, save, close → confirm landing on list; open editor from detail, edit, dirty, close → confirm warning, then confirm + land on detail; load `/edit` URL directly, close → land on detail (fallback) |
| 0005  | Ship | Write `Results/RESULT.md`, `Retro/LESSONS.md`; hv_ship |

## Constraints (carry through to execution)

- UI-only. No backend, DB, or vorch changes.
- Editor must keep working on cold load (`/edit` URL refreshed / bookmarked).
- Close must always land inside `/ansible/*` — defensive guard against malicious `from` state.
- One feature, one PR per affected repo.
- Don't add a custom `ConfirmDialog` if `window.confirm` covers v1 — keep scope tight.
- Sub-nav Playbooks tab must stay active while editing (current behavior, verify with a Playwright snapshot).

## Open questions

These are decisions the implementer should make in Phase 1, NOT in planning:

- **Close button visual treatment**: text button (`Close`) vs icon-only `X` button vs both? Recommend "Close" text button, secondary variant, so screen readers get a clear label without extra `aria-label`.
- **Does a `ConfirmDialog` component already exist?** `grep -rn "ConfirmDialog\|<Confirm" cloud-manager-web/src` before implementing. If yes, use it. If no, `window.confirm` is acceptable.
- **List-row pencil affordance**: clickable on the whole row vs. only the icon area? Recommend only the icon, with `onClick={(e) => e.stopPropagation()}` so the row's existing click-to-open-detail behavior keeps working for the rest of the row.

## Repos in scope

- `cloud-manager-web` — all source changes
- `workflow-exec` — workflow scaffold + Results / LESSONS (mechanical, happens at ship time)

## Out of scope (do NOT touch)

- `RoleFileEditor` internals beyond the new `onDirtyChange` prop
- Role-file editor flow at `/ansible/playbooks/:pid/roles/:roleId`
- Run page, History page
- VM detail page
- `cloud-manager-api`, `vorch-service`, `vorch-lib`
- Database migrations
