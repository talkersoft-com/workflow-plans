# API-PLAN — cloud-manager-api ansible surface

## Problem

The ansible data model is fully deployed but only four entities — `Playbook`, `VmPlaybookAssignment`, `PlaybookRun`, and `FeatureFlag` — have REST controllers (`PlaybookController`, `PlaybookAssignmentController`, `RunController`, `FeatureFlagsController`). Every other ansible-related entity has no API surface at all. Without it, the web UI can't be built, the worker has no materialization endpoint to fetch from, and operators can only manage roles by direct SQL against `clouddb`. This plan specifies the full REST surface needed to close that gap, plus the service-layer hooks that make revisioning automatic and keep `Playbook.argument_schema` in sync with `meta/argument_specs.yml`.

## Existing surface (do not redesign)

| Controller | Routes | Notes |
|---|---|---|
| `PlaybookController` | `GET /api/v1/playbook[/list\|/{id}]`, `POST`, `PATCH /{id}`, `DELETE /{id}` | CRUD on `Playbook` (name, description, content). Already takes `Content` in Register/Update. Needs the revision hook added below. |
| `PlaybookAssignmentController` | `GET /api/v1/virtualmachine/{vmId}/playbook`, `POST` (attach), `PATCH /{asgnId}` (vars override), `POST /{asgnId}/apply`, `DELETE /{asgnId}` | Per-VM assignment + the single-VM apply path that publishes the existing run message. Stays as-is. |
| `RunController` | `GET /api/v1/run/{runId}[/log\|/secret]` | Run read-side + Vault secret reveal. Stays as-is. |
| `FeatureFlagsController` | `GET /api/v1/feature-flags`, `PATCH /api/v1/admin/feature-flags/{key}` | Already serves the in-UI toggle from `WEB-PLAN.md`. No new endpoints needed. |

All four are decorated with `[RequireFeatureFlag("playbooks")]` (except the public `feature-flags` read path). New endpoints follow the same pattern.

## New endpoints

Everything below is gated by `[RequireFeatureFlag("playbooks")]` and returns the standard `{ Message: ... }` error body on failures. Status codes match existing controllers (400, 404, 409, 500). Public IDs on the wire use the prefixes already registered in `EntityPrefixRegistry` — no new prefixes proposed by this plan.

### Ansible roles (global library) — `/api/v1/ansibleRole`

| Verb | Path | Body | Returns | Notes |
|---|---|---|---|---|
| GET | `/api/v1/ansibleRole` | — | `AnsibleRoleDto[]` | List all global roles. |
| GET | `/api/v1/ansibleRole/{id}` | — | `AnsibleRoleDto` | 404 if not found. |
| POST | `/api/v1/ansibleRole` | `{ name, description? }` | `AnsibleRoleDto` | 400 on name collision. 201 on success. |
| PATCH | `/api/v1/ansibleRole/{id}` | `{ name?, description? }` | `AnsibleRoleDto` | Updates metadata only. Files are managed via nested routes. |
| DELETE | `/api/v1/ansibleRole/{id}` | — | 204 | **409 if any `PlaybookGlobalRoleRef` row references this role.** No cascade delete; operator must detach first. |

Service: `IAnsibleRoleService` (new) on top of an `IRepository<AnsibleRole>` (new). DTO: `AnsibleRoleDto { publicId, name, description, created, modified, fileCount }`.

### Role files — `/api/v1/ansibleRole/{roleId}/file`

| Verb | Path | Body | Returns | Notes |
|---|---|---|---|---|
| GET | `/api/v1/ansibleRole/{roleId}/file` | — | `RoleFileSummaryDto[]` | Flat array: `{ publicId, file_path, modified }`. The UI parses `file_path` into a tree. |
| GET | `/api/v1/ansibleRole/{roleId}/file/{fileId}` | — | `RoleFileDto` | Full content. |
| POST | `/api/v1/ansibleRole/{roleId}/file` | `{ file_path, content }` | `RoleFileDto` | 400 on duplicate `file_path` within role, on path containing `..`, on path starting with `/`, or on `content` > 1 MiB. Creates revision_number=1 in same transaction. |
| PATCH | `/api/v1/ansibleRole/{roleId}/file/{fileId}` | `{ content }` | `RoleFileDto` | **Creates new `RoleFileRevision` row in same transaction with `revision_number = max(existing) + 1`.** Validation same as POST. Fires the argument-specs hook (see below). |
| DELETE | `/api/v1/ansibleRole/{roleId}/file/{fileId}` | — | 204 | Cascades revisions. If file was `meta/argument_specs.yml`, fires the argument-specs hook (re-derive with file absent). |

Service: `IRoleFileService` (new). DTOs: `RoleFileSummaryDto`, `RoleFileDto { publicId, role_id, file_path, content, latest_revision_number, modified }`.

### Role file revisions — `/api/v1/ansibleRole/{roleId}/file/{fileId}/revision`

| Verb | Path | Body | Returns | Notes |
|---|---|---|---|---|
| GET | `…/revision` | — | `RoleFileRevisionSummaryDto[]` | Newest first. `{ publicId, revision_number, change_summary, created, created_by }`. |
| GET | `…/revision/{revisionId}` | — | `RoleFileRevisionDto` | Full historic content. |
| POST | `…/revision/{revisionId}/restore` | `{ change_summary? }` | `RoleFileRevisionDto` | Creates a new revision whose content equals the chosen historic revision. Never deletes history. Fires the argument-specs hook. |

Service: `IRoleFileRevisionService` (new).

### Playbook revisions — `/api/v1/playbook/{playbookId}/revision`

Read-only on this controller. Writes happen as a side effect of `PATCH /api/v1/playbook/{id}` when `content` changes (see service-layer hooks below).

| Verb | Path | Returns |
|---|---|---|
| GET | `/api/v1/playbook/{playbookId}/revision` | `PlaybookRevisionSummaryDto[]` newest first |
| GET | `/api/v1/playbook/{playbookId}/revision/{revisionId}` | `PlaybookRevisionDto` (full content) |

Also: `PlaybookController.Update` is modified so any change to `content` writes a `PlaybookRevision` row with `revision_number = max(existing) + 1` in the same EF transaction. See "Service-layer hooks" below.

### Playbook ↔ global role refs — `/api/v1/playbook/{playbookId}/globalRole`

| Verb | Path | Body | Returns | Notes |
|---|---|---|---|---|
| GET | `/api/v1/playbook/{playbookId}/globalRole` | — | `PlaybookGlobalRoleRefDto[]` (joins to `AnsibleRole` for name) | Lists attached global roles. |
| POST | `/api/v1/playbook/{playbookId}/globalRole` | `{ ansibleRoleId }` | `PlaybookGlobalRoleRefDto` | 409 on duplicate. After attach, fires the argument-specs hook to re-derive `Playbook.argument_schema` (the role's argument_specs.yml now contributes). |
| DELETE | `/api/v1/playbook/{playbookId}/globalRole/{refId}` | — | 204 | After detach, fires the argument-specs hook. |

Service: `IPlaybookGlobalRoleRefService` (new).

### Playbook-local roles — `/api/v1/playbook/{playbookId}/role`

Mirrors the global-role shape. A `PlaybookRole` belongs to exactly one playbook; name is unique per playbook.

| Verb | Path | Body | Returns |
|---|---|---|---|
| GET | `/api/v1/playbook/{playbookId}/role` | — | `PlaybookRoleDto[]` |
| GET | `…/{roleId}` | — | `PlaybookRoleDto` |
| POST | `…` | `{ name, description? }` | `PlaybookRoleDto` |
| PATCH | `…/{roleId}` | `{ name?, description? }` | `PlaybookRoleDto` |
| DELETE | `…/{roleId}` | — | 204 (cascades files + revisions) |

Nested file + revision routes are identical in shape to the global versions, just scoped per playbook-local role:

- `/api/v1/playbook/{playbookId}/role/{roleId}/file` — same verbs as `/api/v1/ansibleRole/{roleId}/file`
- `/api/v1/playbook/{playbookId}/role/{roleId}/file/{fileId}/revision` — same verbs as the global equivalent

Revision-creation trigger is the same: PATCH content → new `PlaybookRoleFileRevision`. Argument-specs hook fires for the owning playbook (only) when `meta/argument_specs.yml` changes.

Services: `IPlaybookRoleService`, `IPlaybookRoleFileService`, `IPlaybookRoleFileRevisionService` (all new).

### Playbook run targets — `/api/v1/run/{runId}/target`

Read-only. Worker writes these via internal service calls during execution.

| Verb | Path | Returns |
|---|---|---|
| GET | `/api/v1/run/{runId}/target` | `PlaybookRunTargetSummaryDto[]` — one per target VM, with status, stats summary |
| GET | `/api/v1/run/{runId}/target/{targetId}` | `PlaybookRunTargetDto` — full per-target stats + output |

Service: `IPlaybookRunTargetService` (new). The UI's `AnsibleRunDetailPage` polls this when the parent run is `Running`.

### Multi-VM run trigger — `/api/v1/playbook/{playbookId}/run`

NEW. The schema supports multi-target runs via `PlaybookRunTarget`, but the existing apply path (`PlaybookAssignmentController.Apply`) only handles single-VM assignment-driven runs. This endpoint adds the explicit multi-target path that the `AnsibleRunPage` UI consumes.

| Verb | Path | Body | Returns |
|---|---|---|---|
| POST | `/api/v1/playbook/{playbookId}/run` | `{ vmIds: string[], varsOverride?: object }` | 202 Accepted, body `{ runId }` |

Behavior, in one EF transaction:

1. Validate: `vmIds` non-empty (400); every `vmId` resolves to a known VM (400 with the unknown id); playbook exists and has non-empty `content` (400).
2. Create one `PlaybookRun` row with `status=Queued`, `playbook_id`, `playbook_revision_id` = latest revision id.
3. For each `vmId`:
   - Look up `VmPlaybookAssignment` for (vm, playbook). If found, snapshot `vars_override` onto `PlaybookRunTarget.vars_override_snapshot` and set `assignment_id`.
   - If not found, use the request's `varsOverride` (or `{}`) and leave `assignment_id = null` (ad-hoc target).
   - Insert `PlaybookRunTarget` row with `status=Queued`.
4. Publish a single message `{ runId }` to the `playbook-runs` queue via `IPlaybookRunPublisher`.

Single-VM `POST /api/v1/virtualmachine/{vmId}/playbook/{asgnId}/apply` is preserved unchanged — it's the assignment-driven shortcut. This new endpoint is the explicit-target path.

### Run cancellation — `/api/v1/run/{runId}/cancel`

NEW. The enum has `Cancelled` but no path reaches it today.

| Verb | Path | Body | Returns |
|---|---|---|---|
| POST | `/api/v1/run/{runId}/cancel` | — | 200 with updated `PlaybookRunDto` |

Behavior:

- 409 if `status` is already terminal (`Succeeded`, `Failed`, `Cancelled`).
- If `status == Queued`: flip directly to `Cancelled` in DB, set `completed_at = now()`, ack/no-op for the worker (it'll see the terminal state on next message pull and drop).
- If `status == Running`: publish `{ runId }` to a new `playbook-run-cancel` queue (or fan-out exchange). Worker subscribes to this queue and cancels the matching context; once the subprocess finalizes, worker writes the final `Cancelled` status. The API does NOT flip status to `Cancelled` directly in this case — it leaves the final write to the worker so target stats/output are preserved.

The `playbook-run-cancel` queue is new shared infrastructure introduced jointly by this plan (publisher) and `VORCH-PLAN.md` (subscriber). Both sides need to land together.

## Cross-cutting

- Every new controller carries `[RequireFeatureFlag("playbooks")]`.
- Every new entity gets a DTO under `src/Models/CloudManager.DTO/Models/` and an AutoMapper profile entry under `src/Services/CloudManager.Data.Services/MappingProfiles/`. DTOs surface `public_id` as `publicId` (camelCase on the wire) and never expose `id` (the internal Guid).
- File-path validation, applied identically to `RoleFile.file_path` and `PlaybookRoleFile.file_path`:
  - Reject `..` segments anywhere
  - Reject paths starting with `/` or `~`
  - Reject paths with backslashes
  - Reject `content` byte size > 1 MiB (configurable in `appsettings.json` under `Playbooks:MaxFileBytes`)
- Error model is unchanged from existing controllers: `{ Message: "..." }` with 400 / 404 / 409 / 500.

## Service-layer hooks

These are the non-obvious behaviors that must happen inside EF transactions, not in controller code.

### `PlaybookService.UpdateAsync` — auto-revision on content change

When `PlaybookController.Update` is invoked and the incoming `Content` differs from the stored value:

1. Compute `revision_number = (max(existing PlaybookRevision.revision_number for this playbook) ?? 0) + 1`.
2. Insert a new `PlaybookRevision` row with: `playbook_id`, `revision_number`, `content` = the NEW content, `change_summary` = derived from request (TODO: accept `change_summary` field on `UpdatePlaybookRequest`; default to "edit" if not provided).
3. Update `Playbook.content` and audit fields.
4. Commit the transaction.

If `Content` is unchanged or absent in the PATCH body, no revision row is written.

### `RoleFileService.UpdateAsync` and `PlaybookRoleFileService.UpdateAsync` — auto-revision + argument-specs hook

Identical pattern, with one extra step:

1. Compute `revision_number = (max(existing RoleFileRevision.revision_number for this file) ?? 0) + 1`.
2. Insert the revision row.
3. Update the file row.
4. **If `file_path == "meta/argument_specs.yml"`**:
   - For `RoleFile`: find every playbook with a `PlaybookGlobalRoleRef` pointing at this role's `AnsibleRole`. For each such playbook, invoke `ArgumentSpecsTranslator` to re-derive `argument_schema` from the combined argument_specs across all referenced roles + the playbook's own content. Update `Playbook.argument_schema`.
   - For `PlaybookRoleFile`: only the owning playbook is affected. Same re-derive.
5. Commit the transaction.

The translator already exists at `src/Services/CloudManager.Data.Services/Playbooks/ArgumentSpecsTranslator.cs` and was added for exactly this purpose; this is the first caller.

### Cascade rules

- `AnsibleRole` delete: BLOCKED if any `PlaybookGlobalRoleRef` exists (409 from API). No cascade.
- `RoleFile` delete: cascade `RoleFileRevision`.
- `Playbook` delete (existing endpoint): BLOCKED if any `VmPlaybookAssignment` exists (409 from API, already implemented).
- `PlaybookRole` delete: cascade `PlaybookRoleFile` → `PlaybookRoleFileRevision`.
- `PlaybookGlobalRoleRef` delete: clean detach, fires argument-specs hook.

## Materialization API (for vorch-service)

NEW endpoint that lets the worker fetch a self-contained playbook snapshot in one round trip, instead of crawling files individually.

`GET /api/v1/playbook/{id}/materialized?revision={revisionId}`

Returns:

```json
{
  "playbook_public_id": "pb_8x3k2pm9wq",
  "playbook_revision_id": "pbrev_…",
  "files": [
    { "relative_path": "site.yml", "content": "..." },
    { "relative_path": "roles/postgres/tasks/main.yml", "content": "..." },
    { "relative_path": "roles/postgres/meta/argument_specs.yml", "content": "..." }
  ],
  "secret_inputs": [
    { "extravar_name": "downstream_api_key", "vault_path": "cloudmanager/data/integrations/foo/api_key" }
  ]
}
```

Semantics:

- `files[]` is the full directory tree: the playbook's `content` (rendered as `site.yml` or the path the playbook expects), every file from every referenced global role (under `roles/<role_name>/...`), and every file from every playbook-local role (also under `roles/<role_name>/...` — naming collisions resolved by preferring local roles per ansible convention).
- `revision` query param pins the playbook content version. Defaults to latest. Role files come from each role's CURRENT head; per-run role-revision pinning is deferred and called out in `DATA-MODEL-DELTAS.md`.
- `secret_inputs[]` is computed from every `meta/argument_specs.yml` in the materialized tree. Every option key annotated `x-secret-path: <vault path>` is surfaced here. The API does NOT resolve the values — the worker resolves them with its own Vault token. This keeps the API ignorant of secret material.
- `playbook_public_id` is included for Vault prefix construction (`cloudmanager/vm/instances/{vm_pub}/{pb_pub}/...`) without an extra round trip.

The worker hits this endpoint once per run (in `VORCH-PLAN.md` Per-message flow step 4) and writes everything to a tmpfs workdir.

## OpenAPI / Swagger

All new endpoints documented via XML doc comments on the controller methods so the existing Swashbuckle generation surfaces them in `/swagger`. DTOs annotated with `[JsonPropertyName]` where wire names diverge from C# names.

## Authorization

All routes gated by the existing `[RequireFeatureFlag("playbooks")]` attribute. No per-user RBAC in this plan — deferred. Admin-only routes (currently just `PATCH /api/v1/admin/feature-flags/{key}`) stay as-is; eventually a `[RequireRole("admin")]` attribute will gate them but that's out of scope here.

## Phases

Order from smallest shippable slice to complete feature. Each phase is independently testable.

1. **AnsibleRole + RoleFile CRUD** — no revisions, no argument-specs hook. UI can list and edit role files but loses history on save. Smallest meaningful slice.
2. **Revision write-side** — PATCH on role files auto-creates `RoleFileRevision` rows. Revision read-side routes (GET list + GET by id) added so the UI can show history.
3. **Playbook revisions** — PATCH on `Playbook.content` auto-creates `PlaybookRevision`. Read-side routes added.
4. **PlaybookGlobalRoleRef + argument-specs hook** — `ArgumentSpecsTranslator` wired in. Saving `meta/argument_specs.yml` in a global role propagates to every referencing playbook.
5. **Playbook-local roles** — `PlaybookRole`, `PlaybookRoleFile`, `PlaybookRoleFileRevision` full CRUD. Hook fires only for the owning playbook.
6. **PlaybookRunTarget reads** — `GET …/target[/list/{id}]`. Unblocks the multi-target UI.
7. **Materialization endpoint** — vorch-service can now fetch its run snapshot. Includes `secret_inputs[]` manifest.
8. **Multi-VM run-trigger endpoint** — `POST /api/v1/playbook/{id}/run`. Enables the `AnsibleRunPage` UI.
9. **Run cancellation** — `POST /api/v1/run/{id}/cancel` plus `playbook-run-cancel` queue publisher. Worker side ships from `VORCH-PLAN.md` Phase 7.
