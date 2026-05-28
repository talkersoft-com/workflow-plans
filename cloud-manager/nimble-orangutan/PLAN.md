# Plan: Finish the Ansible run path — Vault scoped tokens, requirement enforcement, multi-controller surface

## Objective

Clear the three follow-ups from lilac-redpanda's LESSONS so the `pg` playbook actually completes against `postgres-test`, so operators can't dispatch a playbook against a controller that's missing its required collections, and so the Collections UI stops silently assuming `hosts[0]` is the controller. End state:

1. The `pg` playbook's `community.hashi_vault.vault_kv2_write` task succeeds because cloud-manager-api mints a Vault token scoped to that run's secret prefix, vorch forwards it to ansible-runner as `VAULT_TOKEN`, and the token is revoked on terminal status.
2. Clicking Run against a playbook whose `playbook_collection_requirements` are not satisfied on the chosen controller returns a clear "missing collection" 400 the UI renders inline — no more "module not found" surprises mid-run.
3. The Collections page (and the new requirements check) honors an explicit controller-host selector backed by `bare_metal.hosts.is_controller`.

## Background

After lilac-redpanda shipped:
- Ansible Collections UI + worker pipeline is live. `community.postgresql` and `community.hashi_vault` install into `/usr/share/ansible/collections/ansible_collections/community/{name}` and the `pg` playbook's modules resolve.
- `playbook_collection_requirements` table exists per DATA-MODEL.md (workflow-plans PR #12). No code reads it yet.
- `cloud-manager-web` picks `hosts[0]` for the Collections page, which works today because there's exactly one controller.

Relevant shipped scaffolding:
- **vorch-lib `vault/` client** writes per-VM SSH keys to `cloudmanager/data/vm/instances/{vm-id}` — same Vault, same KV-v2 mount, same auth model. We extend the same pattern for per-run scoped tokens.
- **`vault-cli cloudmanager`** at `cloud-manager-api/scripts/vault/cli/cloudmanager.py` creates the cloudmanager policy + token. The deployed cloudmanager token already has full read/write on `cloudmanager/*` — it can mint child tokens via `auth/token/create`.
- **`PlaybookRunService.TriggerMultiVmAsync`** is the trigger entry point (`cloud-manager-api/src/Services/CloudManager.Data.Services/PlaybookRunService.cs`). All three features extend this method.
- **`PlaybookRun.VaultSecretsPrefix`** column already exists, currently set to empty string. Becomes the canonical `cloudmanager/data/playbook-secrets/{runId}` path.
- **vorch `playbook_run_handler.fetchRun`** returns the run DTO. Phase A.2 adds `vaultRunToken` to that DTO so the handler can pass it to the exec.

## Design

Three features are independent at the data layer but converge in the UI and in `TriggerMultiVmAsync`. Order them by dependency: A unblocks `pg`'s most pressing failure mode (Vault 403); B catches misconfigurations before dispatch; C is the smallest of the three but a prerequisite for B's host-correct enforcement.

### A. Vault scoped-token mint

#### A.1 — API mints a child token at run trigger

In `PlaybookRunService.TriggerMultiVmAsync`, after the run + targets are persisted but before `_publisher.EnqueueRun(...)`:

```csharp
var prefix = $"cloudmanager/data/playbook-secrets/{run.PublicId}";
var policy = await _vaultClient.WritePolicyAsync($"playbook-run-{run.PublicId}", BuildRunPolicy(run.PublicId));
var child = await _vaultClient.CreateChildTokenAsync(
    policies: new[] { policy.Name },
    ttl: TimeSpan.FromHours(1),
    displayName: $"playbook-run-{run.PublicId}",
    noParent: true,
    renewable: false);

run.VaultSecretsPrefix = prefix;
run.VaultRunToken = child.ClientToken;
await _context.SaveChangesAsync();

_publisher.EnqueueRun(run.PublicId);
```

`BuildRunPolicy` emits exactly:
```hcl
path "cloudmanager/data/playbook-secrets/{publicId}/*" {
  capabilities = ["create", "update", "read"]
}
path "cloudmanager/metadata/playbook-secrets/{publicId}/*" {
  capabilities = ["read", "list", "delete"]
}
```

The `IVaultClient` is a new thin wrapper in `cloud-manager-api/src/Services/CloudManager.Vault.Client/` (new project). Surface: `WritePolicyAsync`, `DeletePolicyAsync`, `CreateChildTokenAsync`, `RevokeTokenAsync`. Wraps the same HTTP calls vorch-lib's `vault/client.go` makes; just C# this time.

Storage: a new `vault_run_token` column (`varchar(256)`, nullable) on `vm.playbook_runs`. Migration in Phase 1.

On any Vault failure here, the API returns **503** (Vault unavailable, try again) — operator-actionable, distinct from a programming-error 500. No AMQP publish if the mint fails. No half-state.

#### A.2 — Run DTO surfaces the token

- Add `string? VaultRunToken` to `DtoModel.PlaybookRun`.
- AutoMapper: `.ForMember(d => d.VaultRunToken, o => o.MapFrom(s => s.VaultRunToken))`.
- **Scrubbed from list endpoints.** `ListByAssignmentAsync` and any list endpoint sets `dto.VaultRunToken = null` before return.
- Only `GetByIdAsync` (the single-fetch endpoint vorch hits) surfaces the real token.
- API logging: the C# `ILogger` gets a custom enricher that redacts any string starting with `hvs.` to `hvs.<REDACTED>`. Belt-and-suspenders.

Wire shape — vorch's `runDTO` in `playbook_run_handler.go` adds:
```go
VaultRunToken string `json:"vaultRunToken,omitempty"`
```

#### A.3 — Vorch forwards token to ansible-runner

In `playbook_run_exec.executeOneTarget` (vorch-service):

```go
env := os.Environ()
if run.VaultRunToken != "" {
    env = append(env, "VAULT_TOKEN="+run.VaultRunToken)
    env = append(env, "VAULT_ADDR="+serverCfg.Vault.Address)
}
cmd.Env = env
```

When `VaultRunToken` is empty (older runs, dev/test), the env vars are simply not set — Ansible's `community.hashi_vault` tasks then fail loudly, which is the correct behavior. No fallback to the long-lived token.

#### A.4 — API revokes on terminal status

New private method on `PlaybookRunService`:

```csharp
private async Task RevokeRunVaultAsync(PlaybookRun run) {
    if (string.IsNullOrEmpty(run.VaultRunToken)) return;
    try {
        await _vaultClient.RevokeTokenAsync(run.VaultRunToken);
        await _vaultClient.DeletePolicyAsync($"playbook-run-{run.PublicId}");
    } catch (Exception ex) {
        _logger.LogWarning(ex, "Vault revoke failed for run {RunId}; sweeper will retry", run.PublicId);
        return;
    }
    run.VaultRunToken = null;
    // VaultSecretsPrefix stays — audit trail of where secrets landed.
}
```

Called from:
- The existing worker-callback PATCH endpoint (`PlaybookRunsController.Patch(...)`) whenever the inbound `status` is `Succeeded`, `Failed`, or `Cancelled`.
- `CancelAsync` when an operator-initiated cancel takes effect.

Idempotent: a failed Vault call leaves `VaultRunToken` set so the next call retries. The sweeper (A.5) finishes whatever leaked.

#### A.5 — Sweeper for orphaned tokens

A `BackgroundService` (`VaultRunTokenSweeper`) registered in `Program.cs`:
- Wakes every 30 minutes.
- `SELECT * FROM vm.playbook_runs WHERE vault_run_token IS NOT NULL AND status NOT IN (Succeeded, Failed, Cancelled) AND started_at < NOW() - INTERVAL '2 hours'`.
- For each: force-mark `status = Failed`, `error_message = "vault-sweep: run exceeded 2h timeout"`, then `RevokeRunVaultAsync(...)`.
- Single-instance API; no leader election needed at the current scale. Documented as a future concern if cloud-manager-api ever HA's.

#### A.6 — Acceptance for (A)

1. Trigger `pg` against `postgres-test`. Inspect the deployed run row: `vault_run_token` is non-null, `vault_secrets_prefix = cloudmanager/data/playbook-secrets/<runId>`.
2. Vorch's exec log shows `VAULT_TOKEN` injected (redacted in log). Ansible's `vault_kv2_write` task writes to `cloudmanager/data/playbook-secrets/<runId>/postgres-test/postgres` successfully.
3. Vault audit log shows the `token/create` at trigger time and `token/revoke` at terminal time.
4. After completion, the same token can no longer write (`curl ... 403`).
5. Force-kill the API mid-run to simulate the orphaned-token case; after 2h + 30m the sweeper revokes it.

### B. playbook_collection_requirements enforcement (refuse-only v1)

#### B.1 — Requirement CRUD endpoints + DTO

- `DtoModel.PlaybookCollectionRequirement` + AutoMapper profile.
- `IPlaybookCollectionRequirementService` with `ListByPlaybookAsync(string playbookPublicId)`, `CreateAsync(...)`, `DeleteAsync(string pcreqPublicId)`.
- New endpoints (under `/api/v1/ansible`):
  - `GET /playbooks/{pid}/collection-requirements` — list
  - `POST /playbooks/{pid}/collection-requirements` — body `{ collectionPublicId, minVersion?, maxVersion? }`. 409 on duplicate `(playbook, collection)`.
  - `DELETE /collection-requirements/{pcreqPublicId}`
- Deriving requirements from playbook YAML's `collections:` block — out of scope.

#### B.2 — Enforcement at TriggerMultiVmAsync

Insert before the run-row persistence in `TriggerMultiVmAsync`:

```csharp
var requirements = await _context.PlaybookCollectionRequirements
    .Where(r => r.PlaybookId == playbook.Id).ToListAsync();
if (requirements.Count > 0) {
    var controllerId = await ResolveControllerHostIdAsync(); // see C
    var installed = await _context.HostAnsibleCollections
        .Where(h => h.HostId == controllerId && h.InstallStatus == CollectionInstallStatus.Installed)
        .ToDictionaryAsync(h => h.CollectionId);
    var missing = new List<MissingRequirement>();
    foreach (var req in requirements) {
        if (!installed.TryGetValue(req.CollectionId, out var hc)) {
            missing.Add(new MissingRequirement(req, reason: "not installed"));
            continue;
        }
        if (!SatisfiesVersion(hc.Version, req.MinVersion, req.MaxVersion)) {
            missing.Add(new MissingRequirement(req, reason: $"version {hc.Version} out of range [{req.MinVersion}, {req.MaxVersion}]"));
        }
    }
    if (missing.Count > 0)
        throw new RequirementsUnmetException(missing);
}
```

`RequirementsUnmetException` is a new typed exception caught by the controller and turned into a `400` with body:
```json
{
  "error": "requirements_unmet",
  "missing": [
    { "collectionId": "acoll_...", "namespace": "community", "name": "postgresql", "minVersion": "4.0.0", "reason": "not installed" }
  ]
}
```

`SatisfiesVersion` uses `Semver.SemVersion.Parse` (Semver NuGet package — already a transitive dep, confirm in Phase 1). For nullable bounds: missing min ≡ "any min", missing max ≡ "any max".

Refuse-only for v1. Auto-install is explicit follow-on.

#### B.3 — UI: PlaybookDetailPage gains "Required collections"

Below the existing Arguments section:
- Panel titled "Required collections".
- Table with columns: `namespace.name`, min version, max version, status badge for the current controller, remove icon.
- "+ Add requirement" button → modal:
  - Select from a dropdown of catalog collections (queried via `api.ansibleCollections.list()`).
  - Min version (optional text input).
  - Max version (optional text input).
- Banner above the panel: red if any requirement is unmet on the selected controller, with a "Fix" link to `/ansible/collections`.

New Redux slice `playbookCollectionRequirementsSlice` with thunks `fetchForPlaybook`, `createRequirement`, `deleteRequirement`.

#### B.4 — UI: AnsibleRunPage shows requirement status

When the user selects a playbook in the Run flow:
- Fetch its requirements (`fetchForPlaybook`).
- Compute missing/satisfied against the selected controller's `host_collections` (already loaded).
- If anything unmet: disable the Run button, render inline `"Run blocked: missing required collection(s): community.postgresql >= 4.0.0"` with a link to Collections.

#### B.5 — Acceptance for (B)

1. Open `/ansible/playbooks/<pg_pid>`. Add `community.postgresql >= 4.0.0`. Row appears, status `Installed` (already on the controller from Flow B of lilac-redpanda).
2. Go to `/ansible/collections`, Remove `community.postgresql`. Back on the playbook detail page: banner appears, status `Not installed` on the requirement row.
3. Try to Run `pg` against `postgres-test`: Run button disabled, inline error names the missing collection.
4. Reinstall `community.postgresql`: banner clears, Run button re-enabled, run dispatches as before.

### C. Multi-controller UI surface (scope-limited)

#### C.1 — Schema

EF Core migration:
- `ALTER TABLE bare_metal.hosts ADD COLUMN is_controller boolean NOT NULL DEFAULT false`.
- Seed in migration body: `UPDATE bare_metal.hosts SET is_controller = true WHERE name = 'ubuntu-server'`.

Entity: add `public bool IsController { get; set; }` to `Host`. Mapper: passthrough. DbContext binding: `entity.Property(e => e.IsController).HasColumnName("is_controller").HasDefaultValue(false).IsRequired();`

#### C.2 — Hosts API

`GET /api/v1/host/list` now returns each host with `isController: bool`. No new endpoint, no query-param filter. UI filters client-side.

The trigger-side `ResolveControllerHostIdAsync()` used by B.2:
- Loads all hosts with `IsController = true`.
- For v1: picks the first one (lexicographic by name). If zero exist, throws `InvalidOperationException("no controller host configured")`.

#### C.3 — UI surface changes

- **`AnsibleCollectionsPage`**: above the catalog table, a `<select>` of controller hosts (filtered by `isController === true`). Default: first by name. Persisted in `localStorage` key `cm.selectedControllerHostId`. Status badges + actions key off the selected host's `host_collections`.
- **`AnsibleCollectionDetailPage`**: same selector. The "Install state on …" section uses the selected host. Plus a compact "All controllers" table below it: one row per controller, showing `version` + `status`. Read-only.
- **`PlaybookDetailPage`** (the new B.3 panel): uses the same persisted-in-localStorage controller selection. If the operator hasn't picked one yet, falls back to first `isController=true`.

#### C.4 — Explicit non-goals

- `playbook_runs` does NOT get `host_id` in this plan. Dispatching runs to a specific controller is a follow-on plan.
- No collection-install fan-out across all controllers. Per-controller install only.
- No controller-pool model.
- The Collections page does NOT render rows-per-(collection, controller) in the main table. One controller at a time via the selector; the multi-controller summary lives on the detail page only.

#### C.5 — Acceptance for (C)

1. After migration applies: `SELECT is_controller, name FROM bare_metal.hosts;` shows `ubuntu-server` as `t`.
2. `/ansible/collections` renders a host selector with `ubuntu-server` in it. Selection persists across reload.
3. Manually `INSERT INTO bare_metal.hosts (..., is_controller) VALUES (..., true)` to add a second controller (with `name='ubuntu-server-staging'`, all other required FKs satisfied). Refresh the page → both hosts in the dropdown. Switching selection refetches the right `host_collections` set.

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000  | Setup | hv_status + hv_init/hv_next; record execution branch |
| 0001  | Migration | EF migration: add `vm.playbook_runs.vault_run_token`; add `bare_metal.hosts.is_controller` + backfill. Apply to clouddb. |
| 0002  | Vault client (C# lib) | New `CloudManager.Vault.Client` project: `IVaultClient` interface + HTTP implementation; DI registration; redact-`hvs.*`-in-logs enricher. Token mint, policy write, token revoke, policy delete. |
| 0003  | API trigger mint + revoke + sweeper (Phase A) | `TriggerMultiVmAsync` mints; `Patch(...)` worker callback revokes on terminal; `CancelAsync` revokes; `VaultRunTokenSweeper` BackgroundService registered in `Program.cs`. DTO field + scrub-from-lists. |
| 0004  | Vorch forwards token (Phase A) | vorch-service `runDTO` adds `vaultRunToken`; `playbook_run_exec.executeOneTarget` sets `VAULT_TOKEN` + `VAULT_ADDR` on the subprocess env. |
| 0005  | Requirements CRUD (Phase B) | DTOs + Service + 3 endpoints + DI. |
| 0006  | Requirements enforcement (Phase B) | `RequirementsUnmetException` + 400 mapping; `SatisfiesVersion`; injection into `TriggerMultiVmAsync` ahead of mint+publish. |
| 0007  | Hosts.IsController (Phase C) | Entity + DbContext + mapper + Hosts DTO + API surface. |
| 0008  | Web: controller selector + slices (Phase C) | `selectedControllerHostId` reducer/persistence; `AnsibleCollectionsPage` + `AnsibleCollectionDetailPage` selector wiring; "All controllers" detail panel. |
| 0009  | Web: Required collections panel (Phase B) | `playbookCollectionRequirementsSlice`; panel on `PlaybookDetailPage`; banner; add-requirement modal. |
| 0010  | Web: Run blocked when requirements unmet (Phase B) | `AnsibleRunPage` requirement check; disabled Run button + inline message. |
| 0011  | Build + deploy | Migration apply; cloud-manager-api + vorch-service + cloud-manager-web installers. All `active`. |
| 0012  | Playwright + Vault audit verification | All acceptance criteria from A.6, B.5, C.5. Capture screenshots. |
| 0013  | Ship | Results + Lessons + hv_ship (5 PRs) |

## Constraints (carry through to execution)

- **camelCase JSON on the wire**, public_ids on the wire, internal UUIDs never leaked. Vault tokens NEVER appear in API logs (the redaction enricher in 0002 enforces this) and NEVER in run output streams.
- **One feature, one PR per affected repo. Five PRs total** — A+B+C land as one bundle per repo, not three.
- **Refuse-only for B in this PR set.** No auto-install — that's a separate plan.
- **`playbook_runs` does NOT get a `host_id` column.** Multi-controller dispatch is a separate plan.
- **The Vault revoke on terminal status must be idempotent.** Multiple worker PATCHes for the same run flipping to Succeeded must not error.
- **No silent fallback to the long-lived cloudmanager token in vorch.** If `vaultRunToken` is empty, env is not set, and any vault task fails loudly.
- **The sweeper marks runs Failed; it does not retry them.** A run whose Vault token expired or got revoked is a dead run; the operator re-triggers.

## Open questions resolved by plan recommendations

- **Vault token TTL:** `1h`. Long enough for any `pg`-sized run; short enough to bound a leak window. Operators with longer playbooks can raise it via config — but a single setting for v1, not per-playbook.
- **Sweeper interval:** `30m`. Stale-run threshold: `2h`.
- **Vault-down at trigger:** API returns **503** (Vault unavailable), not 500. Operator-actionable.
- **Add-requirement form:** dropdown of catalog collections, not free-text. Catalog has ~3 rows today; trivial UX.
- **Banner "Fix" link** on the playbook detail page navigates to `/ansible/collections` with no filter — operator picks from the full list.
- **Detail-page "All controllers" panel** in C: included (recommended), since operators with two controllers want a single glance to see drift.
- **Logger redaction implementation:** custom Serilog enricher (or its built-in destructurer) that matches `hvs\.[A-Za-z0-9_\-]+` and replaces with `hvs.<REDACTED>`. Confirm Serilog is the configured logger in 0002.

## Open questions for the implementer

- **Semver library:** verify `Semver` NuGet is already pulled in transitively. If not, add it (or use a minimal hand-rolled comparator since version specifiers in the wild are mostly `>= X.Y.Z`). Settle in Phase 0006.
- **Run-cancel callback semantics:** if the user cancels mid-flight, the worker may still PATCH terminal status later. Revoke must be idempotent — confirm by writing TC-XXX in Test 0003 that PATCHes the same run twice and asserts no error + token remains revoked.
- **Vault address source for vorch:** `serverCfg.Vault.Address` is the existing field. Confirm it's populated in the deployed config-server.yaml; if not, install-vorch-service.py needs a tweak.

## Repos in scope

- `cloud-manager-api` — migration, entities, DTOs, mapper, services, controllers, new Vault client project, sweeper, DI
- `vorch-lib` — none (runDTO lives in vorch-service)
- `vorch-service` — runDTO field + env-var injection
- `cloud-manager-web` — controller selector + slice; requirements slice + panel + modal; Run-page block; All-controllers detail summary
- `workflow-exec` — workflow scaffold + Results / LESSONS

## Out of scope (explicit)

- Multi-controller dispatch (matching playbook-runs to specific controllers). Separate plan.
- Auto-install of missing requirements at trigger time. Separate plan.
- Synchronized collection install across all controllers.
- Vault SSH-key flow (already shipped under blizzard-sparrow + lilac-redpanda).
- Playbook YAML parsing to derive collection requirements automatically.
- Renewable / re-mintable run tokens. The 1h TTL is the leash.
