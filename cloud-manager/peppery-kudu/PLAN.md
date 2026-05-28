# Plan: Web-driven Ansible collection install / remove / reinstall

## Objective

An operator opens `/ansible/collections`, adds `community.postgresql` to the catalog, clicks Install, and within a minute sees the row flip from Pending → Installing → Installed with the version populated. They can click Remove (row flips Removing → Removed) and Install again to reinstall. Under the hood, cloud-manager-api persists the records, enqueues an AMQP message; vorch-service consumes it, shells out to `ansible-galaxy`, captures output, verifies the install, and PATCHes the run + the host-collection rows back through the API. End state: the `pg` playbook's `community.postgresql.postgresql_db` and `community.hashi_vault.vault_kv2_write` tasks resolve at run time because both collections are installed on the controller.

## Background

### What's already in place

- **Data model** — `planning/workflow-plans/DATA-MODEL.md` (workflow-plans PR #12) defines the four `ansible.*` tables, FK relationships to `bare_metal.hosts` and `vm.playbooks`, and the `CollectionInstallStatus` enum. The migration in Phase 1 below is a direct translation of that doc into EF Core + Postgres SQL.
- **AMQP plumbing pattern** — `playbook-runs` queue ⇄ `PlaybookRunPublisher.cs` ⇄ `messagesub/playbook_run_sub.go` ⇄ `handlers/playbook_run_handler.go`. This is the template. The wire convention (after vorch-lib PR #3 and cloud-manager-api PR #14) is camelCase JSON, `runId` as the body key, public_ids on the wire.
- **Worker callback pattern** — vorch PATCHes back through the API to update run state. Reuse that pattern; don't open a new path.
- **Web sub-nav** — `AnsibleLayout` + `AnsibleSubNav.tsx` shipped under overcast-urchin / cloud-manager-web PR #7. Adding a fifth tab is one entry in the `TABS` array.
- **Public-id convention** — `<prefix>_<Crockford base32>` via `PublicIdInterceptor`. Prefixes are claimed in `EntityPrefixRegistry`; new entities register new prefixes there.

### Why this is the natural next slice

The Run-flow fast-fixes (cloud-manager-api PR #14 + vorch-lib PR #3) made playbook execution work end-to-end up to the actual `ansible-runner` exec. The remaining blocker for `pg` to actually do anything is that `community.postgresql` and `community.hashi_vault` aren't installed on the controller. This feature is the productized path to install them — instead of `sudo ansible-galaxy collection install …` on the box, the operator clicks Install in the UI and the system tracks state.

## Design

### Data layer

The Phase 1 migration creates exactly what DATA-MODEL.md describes — no shape variation. Highlights:

- **`ansible` schema** is new. The first thing the migration does is `CREATE SCHEMA ansible`.
- **`ansible.collections`** — `(namespace, name)` unique. Public-id prefix `acoll_`.
- **`ansible.host_collections`** — `(host_id, collection_id)` unique. `install_status` is the new `CollectionInstallStatus` Postgres enum. `last_install_run_id` is nullable FK to `collection_install_runs`. Public-id prefix `hacoll_`.
  - `ON DELETE CASCADE` from `bare_metal.hosts(id)` — host removal cleans up.
  - `ON DELETE RESTRICT` from `ansible.collections(id)` — can't drop a collection still installed somewhere.
- **`ansible.collection_install_runs`** — `status` reuses the existing `PlaybookRunStatus` enum (int storage). `requested_version` nullable (null = latest). Public-id prefix `acrun_`.
- **`ansible.playbook_collection_requirements`** — `(playbook_id, collection_id)` unique. `min_version`/`max_version` nullable strings (Galaxy-style version specifiers). Public-id prefix `pcreq_`.

**Seed** in the migration: insert two rows into `ansible.collections` for `community.postgresql` and `community.hashi_vault` so the demo path doesn't require the operator to type them in. Comment in the migration explains why.

### API surface

All endpoints under `/api/v1/ansible`. Casing matches existing controllers — kebab-case routes, PascalCase action names.

| Method | Route | Purpose |
|---|---|---|
| `GET` | `/collections` | List catalog rows |
| `POST` | `/collections` | Add to catalog — `{ namespace, name, description?, sourceUrl? }` |
| `GET` | `/collections/{cid}` | Detail + nested `host_collections` rows |
| `DELETE` | `/collections/{cid}` | 409 if any `host_collections` row still references it |
| `GET` | `/hosts/{hid}/collections` | What's installed on this host |
| `POST` | `/hosts/{hid}/collections` | Body `{ collectionPublicId, requestedVersion? }`. Creates `host_collections` row in `Pending`, creates `collection_install_runs` in `Queued`, publishes install AMQP, returns the run public id. 409 if a row already exists unless `?force=true` (reinstall path). |
| `POST` | `/host-collections/{hacollId}/reinstall` | New install run with `--force`. Status returns to `Installing`. |
| `DELETE` | `/host-collections/{hacollId}` | Sets `Removing`, creates removal `collection_install_runs` row, publishes remove AMQP. After worker confirms: status `Removed`, `version` cleared. |
| `GET` | `/collection-install-runs/{cid}` | Status + output |
| `PATCH` | `/collection-install-runs/{cid}` | Worker callback. Body `{ status, output?, errorMessage?, version?, installPath?, completedAt? }`. |

The PATCH endpoint is the worker's writeback path — same shape as the existing `playbook-runs` callback. When `status=Succeeded` and the body includes `version` + `installPath`, the API updates the associated `host_collections` row in the same transaction.

### AMQP — two queues, not one

Two physical queues: `collection-installs` and `collection-removes`. Reasoning: mirrors the existing `playbook-runs` / `playbook-run-cancel` split, lets each handler stay a tight switch-less function, and avoids command-dispatch in the consumer. The cost is one extra queue declaration — negligible.

Publishers in `cloud-manager-api/src/Services/CloudManager.AMQP.Publisher/`:
- `CollectionInstallPublisher` — declares + publishes to `collection-installs`. Body: `{ "runId": "acrun_..." }` (camelCase, per convention).
- `CollectionRemovePublisher` — same for `collection-removes`.

### Worker — vorch-service

Two new consumer goroutines + two new handlers, copy-pasted from the playbook-run pattern then customized.

**`messagesub/collection_install_sub.go`** — same shape as `playbook_run_sub.go`. Decode message → call handler → on error, nack-with-redelivery.

**`handlers/collection_install_handler.go`** flow:
1. `fetchRun(runID)` — GET `/api/v1/ansible/collection-install-runs/{cid}` → returns `{ collectionNamespace, collectionName, requestedVersion, force }`.
2. PATCH run → `status=Running`, `startedAt=now`.
3. Build command:
   ```
   ansible-galaxy collection install <ns>.<name>[:<version>] -p /usr/share/ansible/collections
   ```
   Add `--force` if the run came from `/reinstall` (carry the flag through the API → run row → message).
4. Exec, buffer stdout+stderr together.
5. On non-zero exit → PATCH run `{ status: Failed, output: <buf>, errorMessage: <stderr-tail>, completedAt }`. Return.
6. On zero exit → verify with:
   ```
   ansible-galaxy collection list <ns>.<name> --format json
   ```
   Parse the version + install path.
7. PATCH run `{ status: Succeeded, output: <buf>, version: <parsed>, installPath: <parsed>, completedAt }`. The API uses `version`+`installPath` to update the associated `host_collections` row.

**`handlers/collection_remove_handler.go`** flow:
1. `fetchRun(runID)` → returns `{ collectionNamespace, collectionName, installPath }`.
2. PATCH run `status=Running`.
3. **Important note**: `ansible-galaxy` has no native uninstall. Removal is `rm -rf <installPath>`.
4. **Defensive guard**: verify `installPath` starts with `/usr/share/ansible/collections/ansible_collections/`. If not → PATCH `status=Failed`, `errorMessage="install path failed safety check"`. No filesystem ops.
5. `os.RemoveAll(installPath)`.
6. Verify: run `ansible-galaxy collection list <ns>.<name> --format json`. Expect empty output (or a "not found" line). If the collection still shows up → PATCH `Failed`.
7. PATCH run `Succeeded`. The API updates `host_collections` → `status=Removed`, `version=null`.

**Runtime caveat — `ReadWritePaths`**: the systemd unit currently lists `/kvm-automator /var/lib/libvirt/images /mnt/cloud-storage` as writable. `/usr/share/ansible/collections/` is not in that list. The unit also has `User=root`. Options:
- **A. Widen `ReadWritePaths`** — add `/usr/share/ansible/collections`. Simple, narrowly scoped to the collections path. Implementer's job: edit `install-vorch-service.py` so the regenerated unit includes it. **Recommended for v1.**
- **B. Drop `ProtectSystem=strict`** — too broad, rejected.
- **C. Put collections in `/kvm-automator/ansible-collections/`** — already-writable, but operators would not expect collections there, and the default `ansible-runner` collection-path lookup would have to be overridden via env. **Rejected** — wrong default, confusing for ops.

Plan picks **A**. Phase 4 task explicitly adds `/usr/share/ansible/collections` to `ReadWritePaths` in the installer template.

### Web — `cloud-manager-web`

**Sub-nav** — add `{ label: "Collections", path: "/ansible/collections", match: ["/ansible/collections"] }` as the fifth entry in `AnsibleSubNav.tsx`. No layout changes; the strip already handles N tabs.

**Routes** in `App.tsx`:
- `/ansible/collections` → `AnsibleCollectionsPage` (list)
- `/ansible/collections/:cid` → `AnsibleCollectionDetailPage` (install runs history, action buttons)

The Add form lives in a **modal** triggered from a `+ Add collection` button on the list page — matches the `+ New playbook` pattern on `PlaybooksPage`. No `/new` route (less plumbing, same UX).

**List view** columns:
- `namespace.name`
- description
- latest known version
- **status** badge for the controller host (`bare_metal.hosts` row count is 1 today; show that one). Empty / "not installed" / Pending / Installing / Installed / Failed / Removing / Removed.
- **actions** — column shows different buttons based on the current `host_collections.install_status`:
  - **None**, **Removed**, **Failed** → `Install` button
  - **Installed** → `Remove` + `Reinstall` buttons
  - **Pending**, **Installing**, **Removing** → spinner only (no actions during transition)

**Detail view**:
- Collection metadata at the top
- Table of install runs (most recent first): started, completed, status, version, button to expand output
- The same action buttons as the list row

**Redux slices**:
- `collectionsSlice` — catalog list/get/create/delete
- `hostCollectionsSlice` — per-host install records, install/remove/reinstall actions
- `collectionInstallRunsSlice` — run rows + polling thunk for `Installing` / `Removing` states

Polling: while a visible row is in `Installing` or `Removing`, poll `GET /collection-install-runs/{id}` every 5 s. Stop on terminal status. Same pattern as the existing playbook-run polling if any exists; otherwise this is the precedent.

### Lifecycle examples

**Install path**:
```
UI button → POST /hosts/{hid}/collections { collectionPublicId }
API → INSERT host_collections (Pending) + INSERT collection_install_runs (Queued)
API → CollectionInstallPublisher.Publish({ runId })
vorch consumes → PATCH run (Running)
vorch execs ansible-galaxy collection install … → captures output
vorch PATCH run (Succeeded, version, installPath)
API → UPDATE host_collections (status=Installed, version, install_path, last_install_run_id)
UI poll picks up Installed; row badge updates
```

**Remove path**:
```
UI button → DELETE /host-collections/{hacollId}
API → UPDATE host_collections (status=Removing) + INSERT collection_install_runs (Queued, removal)
API → CollectionRemovePublisher.Publish({ runId })
vorch consumes → PATCH run (Running)
vorch validates install_path prefix → os.RemoveAll(install_path)
vorch verifies via ansible-galaxy collection list
vorch PATCH run (Succeeded)
API → UPDATE host_collections (status=Removed, version=null)
UI poll picks up Removed
```

**Reinstall path** = `POST /host-collections/{hacollId}/reinstall`. Same as install but `--force` on the ansible-galaxy command. Status flow: `Installed` → `Installing` → `Installed`. Idempotent.

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000  | Setup | hv_status + hv_init/hv_next; record execution branch |
| 0001  | DB migration | EF Core migration: new `ansible` schema, four tables, `CollectionInstallStatus` enum, indexes, seed of `community.postgresql` + `community.hashi_vault` |
| 0002  | API entities + DTOs + AutoMapper | Entity classes, DTOs, mapping profile, public-id prefix registrations, DbContext bindings, service interfaces |
| 0003  | API endpoints | All ten endpoints listed in the Design section, with the request-shape DTOs and validation |
| 0004  | API publishers | `CollectionInstallPublisher`, `CollectionRemovePublisher`, queue declarations, registration in DI |
| 0005  | vorch-lib messages | `CollectionInstallRunMessage` + `CollectionRemoveRunMessage` with camelCase JSON tags |
| 0006  | vorch-service consumers + handlers | Two subscribers + two handlers; widen `ReadWritePaths` in `install-vorch-service.py` |
| 0007  | Web Collections tab | `AnsibleSubNav` entry, two pages, three Redux slices, modal Add form, polling |
| 0008  | Build + deploy | DB migration apply; cloud-manager-api build + install-api-service.py; vorch-service build + install-vorch-service.py; cloud-manager-web build + install-web-app.py |
| 0009  | Verify e2e (Playwright) | Walk all six acceptance criteria from the requirements; collect screenshots + DB row state at each transition |
| 0010  | Ship | Results + Lessons + hv_ship |

## Open questions resolved by plan recommendations

These are the calls the requirements left open; the plan answers them so the implementer doesn't have to:

- **Two AMQP queues** (install + remove), not one with a command field — mirrors playbook-runs / playbook-run-cancel.
- **Synchronous output capture** (full buffer on completion), not chunked streaming. Install runs are ~10–30 s; chunking adds complexity for no observed benefit. Revisit if a collection takes minutes.
- **Seed `community.postgresql` + `community.hashi_vault` in the migration** — comment in the migration explains it's for the `pg` demo path.
- **Add form is a modal** on the list page, not a dedicated route. Matches `PlaybooksPage`.
- **Reinstall uses `--force`**, not a Remove-then-Install two-step. One atomic run, faster, idempotent.
- **`ReadWritePaths` widens to include `/usr/share/ansible/collections`** (Option A in the design). Edit `install-vorch-service.py` template accordingly.

## Constraints (carry through to execution)

- camelCase JSON on the wire. public_ids on the wire. internal UUIDs never leaked.
- One bug or feature, one PR per affected repo. Five PRs expected.
- Don't reuse the `playbook-runs` queue. Don't reuse `PlaybookRunStatus` for `host_collections.install_status` (that's the lifecycle enum, distinct from run status).
- Worker writes are PATCHes through the API — vorch never opens a DB connection.
- Don't implement `playbook_collection_requirements` enforcement in this PR set. Schema exists; reading + enforcing it during a playbook run is a separate plan.
- Don't widen `ReadWritePaths` beyond `/usr/share/ansible/collections` — keep the security profile tight.

## Non-goals (do NOT do)

- Multi-controller. `host_id` is in the schema and joined correctly; UI shows the one controller for now.
- Galaxy API integration (calling galaxy.ansible.com to fetch description / latest_known_version). Manual entry only.
- Private collection registries / Galaxy mirrors.
- Per-target-VM collection scoping (collections live on the controller, not the targets).
- A custom uninstall implementation that doesn't go through `rm -rf` — there is no official `ansible-galaxy uninstall`. Future improvements (Vault audit trail, backup snapshot before delete) are out of scope.

## Repos in scope

- `cloud-manager-api` — migration, entities, DTOs, AutoMapper, endpoints, publishers
- `vorch-lib` — message structs (one PR with two struct additions)
- `vorch-service` — two subscribers + two handlers + installer env tweak
- `cloud-manager-web` — sub-nav tab + two pages + three slices + modal
- `workflow-exec` — workflow scaffold + Results / LESSONS at ship time
