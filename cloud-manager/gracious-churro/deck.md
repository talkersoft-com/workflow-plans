# Deck: cloud-manager

## Deck name
`cloud-manager`

## Repos with code changes

| Repo | What changes |
|------|-------------|
| `cloud-manager-api` | Add `PagedResult<T>` DTO. Add `PlaybookRunService.ListAsync(vmPublicId?, playbookPublicId?, status?, page, pageSize)` (joins `playbook_run_targets` for `vmId` filter; scrubs `VaultRunToken`; clamps pageSize 1..200). Add `[HttpGet] List` action on `RunController` exposing `GET /api/v1/Run?vmId&playbookId&status&page&pageSize`. |
| `cloud-manager-mcp` | Add `cloud_playbook_run_list` tool — thin pass-through over `GET /api/v1/Run`. Args: `vmId?`, `playbookId?`, `status?`, `page?`, `pageSize?`. Returns the API envelope verbatim. |
| `cloud-manager-web` | Replace `AnsibleHistoryPage` EmptyState with a paginated 50/page grid bound to the new endpoint. Add `RunHistoryGrid`, `VmFilterCombobox`, `Pagination` components. Extend `ansibleRunsSlice` with `page`/`pageSize` and URL ↔ slice sync. Each row links to `/ansible/runs/:runId`. |

Repos not listed will be on the feature branch but skipped by `hv_ship`.

## Branch

`gracious-churro`

## Initialize (Task 0000)

The deck is already on `gracious-churro` from `hv_next`. Confirm:

```
hv_status  deck: "cloud-manager"
```

If somehow not on `gracious-churro`:

```
hv_next  deck: "cloud-manager"
```

## Ship

```
hv_ship  deck: "cloud-manager"
         message: "ansible history: paginated runs grid with VM filter; new GET /api/v1/Run + cloud_playbook_run_list MCP tool"
         title:   "ansible history: paginated playbook-run grid with VM filter"
```
