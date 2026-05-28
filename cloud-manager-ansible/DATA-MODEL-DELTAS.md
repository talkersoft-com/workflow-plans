# DATA-MODEL-DELTAS — schema requirements derived from the other plans

## Problem

`../DATA-MODEL.md` describes the static schema as deployed in `clouddb`. The API, web, and vorch plans in this bundle surface a small number of additional column / index needs against that schema. This file enumerates them as a single migration shopping list with rationale so we don't sprinkle migration thinking across three plans — and so reviewers can see exactly what changes (if any) are required before any of the implementation workflows can ship cleanly.

## Audit of derived needs

### From API-PLAN

**Does `Playbook` need an `active_revision_id` FK to pin "the current revision shown in the editor"?**

**Decide: SKIP.** `max(PlaybookRevision.revision_number for this playbook)` already answers "what's the latest revision" with no extra column. The "active revision" concept is the latest revision, full stop — there's no use case in any of the plans for pinning an older revision as "active". If a future need arises (e.g. "rollback this playbook to revision N as the new live version"), the rollback operation just creates a NEW revision (revision_number = max + 1) whose content equals the historic one. Same restore pattern as `RoleFileRevision`. No schema change.

**Does `RoleFile` need a unique index on `(role_id, file_path)` to enforce no duplicate paths within a role?**

**Decide: ADD.** The API's POST `/api/v1/ansibleRole/{roleId}/file` validates "no duplicate file_path within role" at the service layer (400 on collision). Without a DB-side unique constraint, a concurrent double-POST race could produce two rows with the same `(role_id, file_path)` and corrupt the materialization output (which file does the worker write to disk?). The unique index makes the constraint authoritative and lets EF surface the violation as a `DbUpdateException` the service translates to 400.

**Same question for `PlaybookRoleFile` on `(playbook_role_id, file_path)`?**

**Decide: ADD.** Identical reasoning.

**Does the revisioning need an explicit "current revision pointer" or is `max(revision_number)` sufficient?**

**Decide: max() is sufficient.** Already covered above. No pointer column.

**Does `AnsibleRole` need a `deleted_at` timestamptz for soft-delete, since revision history would otherwise be orphaned?**

**Decide: SKIP.** API-PLAN already blocks `AnsibleRole` deletion with 409 if any `PlaybookGlobalRoleRef` references it. So the only way to actually delete a role is to first detach it from every playbook — at which point the history is orphaned only in the sense that no playbook depends on it. Hard delete is acceptable here; if someone wants to preserve the role's file history after detaching everywhere, the workaround is to rename the role to e.g. `archive_<name>` and leave it in place. Adding soft-delete would force every query path to filter on `deleted_at IS NULL`, which is a pervasive cost for a rare operation.

### From WEB-PLAN

**Does the file tree need a `RoleFile.parent_path` (or similar) to render fast, or is parsing `file_path` on the fly fine?**

**Decide: SKIP.** Even a large role rarely has more than ~50 files. Client-side tree-building from a flat array of `file_path` strings is microseconds — not a perf concern. Adding `parent_path` would require keeping it in sync on every PATCH (since file paths are immutable for the same row in this design — moving a file means deleting + creating), which is more code surface than the savings justify.

**Does the revision drawer need a `RoleFileRevision.diff_summary` text column, or can the UI compute diffs client-side from content?**

**Decide: SKIP.** Monaco's built-in diff view computes diffs client-side from two strings instantly. A `diff_summary` column would be denormalized data that the server has to keep correct on every write — extra failure mode for zero UX gain.

### From VORCH-PLAN

**Does `PlaybookRun` need a `log_path` text column to point at the on-disk log file?**

**Decide: SKIP.** The worker writes to `${PLAYBOOK_LOG_DIR}/<runId>.log` — the path is fully derivable from the public_id and the env var. A column would just duplicate that derivation. The existing `output` text column is the wrong place for multi-MiB logs (would balloon the row), but `PLAYBOOK-INTEGRATION-PLAN.md` already documents the trade-off: keep a size-capped tail in `output` for "last 200 lines" preview, full log on disk. No new column needed.

**Does `PlaybookRunTarget` need `started_at` and `completed_at` columns (currently only the parent `PlaybookRun` has them)?**

**Decide: ADD.** The UI's `AnsibleRunDetailPage` shows per-target duration in `RunTargetsList`. Without per-target timestamps, duration can only be computed for the whole run, which is misleading for multi-target runs where targets execute sequentially. The `Audit` base class gives `created` (≈ enqueue time) and `modified` (≈ last-status-update time), but those aren't semantically the same as `started_at` (worker began executing this target's subprocess) and `completed_at` (subprocess exited). The worker already sets these conceptually in VORCH-PLAN step 7e and 7h; making them columns just persists what the worker is already tracking.

**Does the queue-message contract need a corresponding column like `PlaybookRun.queued_message_id` for dedup on redelivery?**

**Decide: SKIP.** VORCH-PLAN's reconcile (step 2 of per-message flow) checks `PlaybookRun.status` before re-executing. If status is already terminal, ack and drop. That's sufficient idempotency — the message ID would add a second redundant check. RabbitMQ at-least-once + worker reconcile is a standard pattern and works fine without a dedup column.

**Does `PlaybookRun` need a `worker_id` text column for multi-worker debugging?**

**Decide: SKIP for v1, revisit later.** Useful for ops once we run multiple workers (which-worker-did-this question), but v1 runs a single worker. Adding it now means populating it on every status write for no immediate value. Mark as a follow-on plan item when we scale to multiple worker instances.

**Does the worker need a `PlaybookRun.cancel_requested_at` column for the cancel signal?**

**Decide: SKIP.** VORCH-PLAN uses an in-process map `runID → context.CancelFunc` plus the new `playbook-run-cancel` queue, not a DB poll. The status enum's `Cancelled` value (a valid target state from `Running`) suffices for the persisted outcome. The "cancel requested vs cancel completed" distinction only matters in the worker's in-memory state machine, not in the DB — once the worker writes `status = Cancelled`, the cancel is complete.

**`Audit.created_by` on `PlaybookRun` — populated correctly by `RunController.Apply` and the new multi-VM run endpoint?**

**Decide: VERIFY, no schema change.** The column already exists via the `Audit` base class. The acceptance criterion is that both single-VM apply (via `PlaybookAssignmentController.Apply`) and multi-VM trigger (via the new `POST /api/v1/playbook/{id}/run`) write the authenticated user (or a placeholder until auth is wired) into `created_by` so the history page can answer "who triggered this." Both controllers already get `created` via the audit pipeline; if `created_by` is `""` today, that's a pipeline bug to fix in the API implementation, not a schema delta.

## Migration shopping list

Three discrete changes, all grouped into one migration `AddAnsibleConstraintsAndTargetTimestamps` to keep the diff small and the rollback story simple.

### 1. Unique index on `vm.role_files (role_id, file_path)`

- Schema / table: `vm.role_files`
- Index name: `ix_role_files_role_id_file_path_unique`
- Columns: `(role_id, file_path)`, UNIQUE
- EF entity: `RoleFile.cs`, configured in `CloudManagerDbContext.OnModelCreating` via `entity.HasIndex(e => new { e.RoleId, e.FilePath }).IsUnique();`
- Rationale: enforces no-duplicate-path-within-role at the DB layer; lets the service translate the unique violation to 400.

### 2. Unique index on `vm.playbook_role_files (playbook_role_id, file_path)`

- Schema / table: `vm.playbook_role_files`
- Index name: `ix_playbook_role_files_playbook_role_id_file_path_unique`
- Columns: `(playbook_role_id, file_path)`, UNIQUE
- EF entity: `PlaybookRoleFile.cs`, same `HasIndex` config pattern
- Rationale: same as above, for playbook-local role files.

### 3. Add `started_at` and `completed_at` to `vm.playbook_run_targets`

- Schema / table: `vm.playbook_run_targets`
- Columns:
  - `started_at timestamptz NULL` (NULL until the worker begins executing this target)
  - `completed_at timestamptz NULL` (NULL until the subprocess exits or the target is cancelled)
- EF entity: `PlaybookRunTarget.cs` — add `public DateTime? StartedAt { get; set; }` and `public DateTime? CompletedAt { get; set; }`
- DbContext: default-column mappings via the existing snake_case convention; no extra config needed.
- Rationale: per-target duration display in the UI; persisted state the worker already tracks in memory.

Single migration name: `AddAnsibleConstraintsAndTargetTimestamps`.

## EntityPrefixRegistry additions

**None.** No new entity types are introduced by this delta list. The existing prefixes (`pb`, `pbrev`, `ansr`, `rf`, `rfrev`, `pgr`, `plr`, `plrf`, `plrfr`, `prt`, `asgn`, `run`, `ff`) cover everything the API, web, and vorch plans reference. Confirmed no collisions.

## Backfill plan

- **Unique indexes** (changes 1 and 2): no backfill needed. Both tables are empty in production today (the feature flag has never been turned on, so no operator has created roles or role files). If for some reason there were pre-existing duplicate rows, the index creation would fail at migration time and surface them; we'd resolve them manually before retrying. Verify pre-migration with `SELECT role_id, file_path, COUNT(*) FROM vm.role_files GROUP BY 1, 2 HAVING COUNT(*) > 1;` (and the analog for `playbook_role_files`).
- **Timestamp columns** (change 3): nullable on add, so no backfill required. Existing `playbook_run_targets` rows (none in production, but in case) keep NULL `started_at` / `completed_at` and are interpreted by the UI as "unknown duration" rather than "0 duration".

## Rollback notes

All three changes are reversible without data loss:

- Dropping the unique indexes is safe — drops constraint, keeps rows.
- Dropping the timestamp columns loses the per-target timing data captured between migration and rollback, but no other table depends on those columns, so it's a clean drop.

Migration `Down()` should explicitly do both (drop indexes, drop columns) so EF can roll back cleanly.

## Summary

- **3 schema changes**, all in one migration (`AddAnsibleConstraintsAndTargetTimestamps`).
- **0 new entities**, **0 new EntityPrefixRegistry entries**.
- **Cheap and reversible** — should land before API-PLAN Phase 1 ships, but won't block any of the surface plans from being written or reviewed in parallel.
