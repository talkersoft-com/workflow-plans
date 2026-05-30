# Plan: paginated playbook run history with VM filter

## Plan overview

This plan turns the stubbed `/ansible/history` page into a real paginated browse surface — 50 runs per page, ordered newest-first, filterable by VM — that drains from the existing `vm.playbook_runs` audit table. The live run page at `/ansible/runs/:runId` is unchanged: both the Run button and History grid rows route there. When complete, an operator can land on `/ansible/history`, narrow to a specific VM via a combobox, page through every run that ever targeted it, and deep-link straight into the per-run output. The paginated runs-list endpoint that the existing AnsibleHistoryPage stub already names as FIX-001 lands first; MCP and Web bind to it after.

## Objective

Replace `AnsibleHistoryPage`'s "Run listing endpoint pending" EmptyState with a paginated 50/page grid backed by a new `GET /api/v1/Run` list endpoint. The grid filters by VM, page reflects in the URL (deep-linkable), and rows link to `/ansible/runs/:runId`. List responses scrub `VaultRunToken` — the existing rule that only `GetByIdAsync` surfaces the real token stays inviolate.

## Background

The audit data has been there all along — `vm.playbook_runs` and `vm.playbook_run_targets` capture status, timing, stats, output, vars snapshot, secrets prefix, and per-target VM linkage. The live `RunController` (`/api/v1/Run/{runId}` + `/log` + `/cancel`) covers per-run reads, but there is no list endpoint, so the History page that was scaffolded for this purpose just shows an EmptyState (`cloud-manager-web/src/pages/AnsibleHistoryPage.tsx:22-26`, comment cites FIX-001).

The Redux slice (`ansibleRunsSlice.ts`) and filter component (`RunHistoryFilters.tsx`) already anticipate this shape — the slice reserves `vmFilter?: string` and `playbookFilter?: string`. The work is filling in what was scaffolded, not designing it from scratch.

The operator confirmed two trade-off calls up front:

1. **vmId semantics = "any target matches"** — a run is "for VM X" if any of its `playbook_run_targets` rows has `VmId = X`. Runs are inherently multi-target so this is the broadest useful semantic. The narrower "primary target only" alternative was rejected.
2. **Filters live in the URL** — the History view is bookmarkable and shareable; URL params are the source of truth and Redux mirrors them. Consistent with the deep-link note already in `ansibleRunsSlice.ts`.

## Design

### Phase 1 — Service-layer list with filters + paging

`cloud-manager-api/src/Services/CloudManager.Data.Services/PlaybookRunService.cs` gains `ListAsync`, sibling to the existing `ListByAssignmentAsync`:

```csharp
// Interface
Task<PagedResult<DtoModel.PlaybookRun>> ListAsync(
    string? vmPublicId,
    string? playbookPublicId,
    PlaybookRunStatus? status,
    int page,
    int pageSize);
```

```csharp
// PagedResult — new envelope in CloudManager.DTO.Models
public record PagedResult<T>(IEnumerable<T> Items, int Page, int PageSize, int Total);
```

```csharp
// Body sketch
page     = Math.Max(1, page);
pageSize = Math.Clamp(pageSize, 1, 200);

var q = _context.PlaybookRuns.AsNoTracking();

if (!string.IsNullOrWhiteSpace(playbookPublicId))
{
    var pbGuid = await _context.Playbooks.GuidByPublicIdAsync(playbookPublicId);
    q = q.Where(r => r.PlaybookId == pbGuid);
}
if (status.HasValue)
{
    q = q.Where(r => r.Status == status.Value);
}
if (!string.IsNullOrWhiteSpace(vmPublicId))
{
    var vmGuid = await _context.VirtualMachines
        .Where(v => v.PublicId == vmPublicId)
        .Select(v => (Guid?)v.Id).FirstOrDefaultAsync();
    if (vmGuid is null) return new PagedResult<DtoModel.PlaybookRun>(Array.Empty<DtoModel.PlaybookRun>(), page, pageSize, 0);
    q = q.Where(r => _context.PlaybookRunTargets.Any(t => t.RunId == r.Id && t.VmId == vmGuid.Value));
}

var total = await q.CountAsync();
var rows  = await q.OrderByDescending(r => r.StartedAt)
                   .Skip((page - 1) * pageSize)
                   .Take(pageSize)
                   .ToListAsync();

// Resolve playbook public_ids in one batch (avoid N+1)
var pbIds = rows.Select(r => r.PlaybookId).Distinct().ToList();
var pbMap = await _context.Playbooks.AsNoTracking()
    .Where(p => pbIds.Contains(p.Id))
    .ToDictionaryAsync(p => p.Id, p => p.PublicId);

var items = rows.Select(r =>
{
    var dto = _mapper.Map<DtoModel.PlaybookRun>(r);
    dto.PlaybookId    = pbMap.TryGetValue(r.PlaybookId, out var pid) ? pid : r.PlaybookId.ToString();
    dto.VaultRunToken = null; // scrub — only GetByIdAsync surfaces the token
    return dto;
}).ToList();

return new PagedResult<DtoModel.PlaybookRun>(items, page, pageSize, total);
```

The `vaultRunToken` scrub is non-negotiable — the same line exists in `ListByAssignmentAsync` and must exist here.

### Phase 2 — Controller endpoint

`cloud-manager-api/src/CloudManager.API/Controllers/RunController.cs` gains a list action:

```csharp
[HttpGet]
public async Task<IActionResult> List(
    [FromQuery] string? vmId,
    [FromQuery] string? playbookId,
    [FromQuery] PlaybookRunStatus? status,
    [FromQuery] int page = 1,
    [FromQuery] int pageSize = 50)
{
    try
    {
        var result = await _service.ListAsync(vmId, playbookId, status, page, pageSize);
        return Ok(result);
    }
    catch (Exception ex) { _logger.LogError(ex, "list failed"); return StatusCode(500, new { Message = "Internal server error" }); }
}
```

Gated by the existing `[RequireFeatureFlag("playbooks")]`. camelCase JSON on the wire (controlled by existing serializer settings). Public IDs only — `vmId`, `playbookId` accept `vm_XXXX` / `pb_XXXX` strings, never UUIDs.

### Phase 3 — MCP pass-through

`cloud-manager-mcp/src/tools/run.ts` (or whichever module owns `cloud_run_get` / `cloud_run_logs`) gains:

```typescript
// Before
// cloud_run_get(runId)
// cloud_run_logs(runId)
// cloud_run_cancel(runId)

// After
// cloud_run_get(runId)
// cloud_run_logs(runId)
// cloud_run_cancel(runId)
// cloud_playbook_run_list({ vmId?, playbookId?, status?, page?, pageSize? })
```

The new tool issues `GET /api/v1/Run` with the args as query params, returns the envelope verbatim. Default `pageSize` left to the API (50).

### Phase 4 — Web grid + URL-driven filter state

`cloud-manager-web/src/pages/AnsibleHistoryPage.tsx` replaces the `EmptyState` with a real grid. New supporting pieces:

- `src/api/runs.ts` (or extend existing api wrapper) — `listRuns(query: { vmId?; playbookId?; status?; page?; pageSize? })`.
- `src/components/RunHistoryGrid/RunHistoryGrid.tsx` — table rows, status badge, playbook name, target VM chip list (from `playbook_run_targets`), startedAt / completedAt / duration, click handler that navigates to `/ansible/runs/:runId`.
- `src/components/VmFilterCombobox/VmFilterCombobox.tsx` — debounced combobox sourced from `cloud_vm_list` (or its web-client wrapper).
- `src/components/Pagination/Pagination.tsx` — prev / next / "page N of M". Reused across other lists later.
- `ansibleRunsSlice.ts` — extend `FilterState` with `page?: number` and `pageSize?: number`. Mirror URL → slice on mount + on URL change.

URL contract:

```
# Before
/ansible/history

# After
/ansible/history?vmId=vm_GY68JPPKDN&page=2&pageSize=50
```

The existing `RunHistoryFilters` "by playbook / by VM" mode toggle is preserved as a grouping affordance — it controls which column gets visual emphasis but does NOT change the API call.

### Phase 5 — E2E verify

- `curl -sS http://localhost:5250/api/v1/Run?pageSize=50` returns up to 50 items, ordered StartedAt DESC, with a numeric `total` field. No item has a non-null `vaultRunToken`.
- `curl -sS http://localhost:5250/api/v1/Run?vmId=vm_XXXX` returns only runs that targeted that VM.
- `cloud_playbook_run_list` resolves the same shape over MCP.
- Visiting `/ansible/history` shows a grid of 50 rows; selecting a VM filters; URL updates to `?vmId=...&page=1`; clicking any row navigates to `/ansible/runs/:runId` and shows the existing live/historical run detail.

## Implementation phases

| Phase | Title                                  | Description |
|-------|----------------------------------------|-------------|
| 0000  | Setup                                  | hv_status; confirm `gracious-churro` across all repos |
| 0001  | API: ListAsync service method          | Add `PlaybookRunService.ListAsync` + `PagedResult<T>` DTO; filter by vmId / playbookId / status; clamp pageSize 1..200; scrub VaultRunToken |
| 0002  | API: GET /api/v1/Run controller action | Wire query params → service; preserve `[RequireFeatureFlag("playbooks")]` |
| 0003  | MCP: cloud_playbook_run_list tool      | Thin pass-through over the new API endpoint |
| 0004  | Web: AnsibleHistoryPage grid           | Replace EmptyState with paginated grid; add VM filter combobox sourced from cloud_vm_list; URL-driven filter state mirrored into Redux; rows link to /ansible/runs/:runId |
| 0005  | E2E verify                             | curl the API + load /ansible/history in a browser; filter by VM; page; click a row; confirm run detail loads |
| 0006  | Write Results/RESULT.md + Retro/LESSONS.md; hv_ship |

## Open questions

- **Pagination beyond 200/page** — should the cap be 200, or should there be a `pageSize=all` escape hatch for ops investigations? Proposal: hard cap 200, no escape hatch; ops can scroll pages.
- **`status` query param value format** — accept the enum integer (matches DB) or the case-insensitive string ("Queued" / "Running" / ...)? Proposal: accept both via ASP.NET model binding's default enum converter.
- **Target VMs column rendering** — show all target VM chips inline, or count + "click to expand"? Proposal: show up to 3 chips inline with "+N more" overflow.
