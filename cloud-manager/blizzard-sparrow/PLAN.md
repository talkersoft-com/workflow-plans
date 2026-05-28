# Plan: Fix VM-creation error in cloud-manager

## Objective

Operator gets an error when creating a VM via cloud-manager. Reproduce the failure end-to-end (browser → cloud-manager-api → vorch_service → libvirt), capture the actual error message + HTTP status + stack trace, identify the root cause with file/line precision, and ship a minimal targeted fix to the layer(s) actually responsible. End state: creating a VM from `/create` succeeds without error in the deployed environment on `ubuntu-server.talkersoft.com`.

The exact error is **not yet captured**. The first task in execution reproduces it; this plan is structured so the diagnosis-and-fix tasks read the captured artifacts and update their own definitions of done with the actual file/line references once known.

## Background

The VM-creation flow today:

1. **`cloud-manager-web/src/pages/CreateVmPage.tsx`** — user fills name, host, image, memory, disk, description; submit fires `api.vms.create(hostId, ...)`.
2. **`cloud-manager-web/src/services/api.ts`** `vms.create` POSTs to `/api/v1/virtualmachine/create/{hostId}` with body `{ vmi: imageId, virtualMachine: { name, description, memory, diskSizeGb } }`.
3. **`cloud-manager-api`** `VirtualMachineController.Create(hostId, body)` (file: `src/CloudManager.API/Controllers/VirtualMachineController.cs`) — persists the VM row, publishes a `create-vm` command on the `OrchestrationQueue` exchange via `CloudManager.AMQP.Publisher`, returns `{ virtualMachineId }`.
4. **`vorch_service`** subscribes to `OrchestrationQueue` via `messagesub.StartConsumer` (file: `vorch-service/vorch-service/messagesub/messagesub.go`), case `"create-vm"` dispatches to `handlers.CreateVMCommand` (file: `vorch-service/vorch-service/handlers/vm_handlers.go`).
5. **`vorch-lib`** `provision.InitializeHost(...)` (in `vorch-lib/provision/`) does the libvirt + qemu-img + virt-install work on the host.

Recent changes that touched this path:
- WEB Phase 3 (PR #6): deleted `VmPlaybooks` component + its render on `VmDetailPage.tsx`. Not on the CREATE path but worth ruling out.
- vorch-service PR #7: `go mod tidy` cleanup (no logic changes).
- All four code services were rebuilt and redeployed during the overcast-urchin work; the API and vorch binaries at `/opt/cloud-manager-api` and `/kvm-automator` are current.
- The clouddb migration `AddAnsibleConstraintsAndTargetTimestamps` was applied directly via `dotnet ef database update` (not via `scripts/database/deploy-database.py`). VM-related tables (`vm.virtual_machines`, `vm.virtual_machine_images`, `vm.hosts`) were NOT touched by that migration but are worth verifying haven't drifted.

Likely failure surfaces, ranked by base rate of VM-create issues in this codebase:

1. **API request shape mismatch** — `CreateVmPage` form fields vs `vms.create` payload vs controller's bound model. The API client uses `{ vmi, virtualMachine: { name, description, memory, diskSizeGb } }` — verify `diskSizeGb` actually exists on the controller's request DTO and matches the entity's `disk_size_gb` column.
2. **AMQP publish failure** — if RabbitMQ creds rotated or the queue config drifted.
3. **vorch handler crash** — `provision.InitializeHost` parameters could have changed; the Go handler signature in `vm_handlers.go:33-55` passes a specific arg order; if vorch-lib changed any parameter type/order, the build catches it, but logic errors (e.g. wrong default for `osTag`) don't.
4. **libvirt / virt-install** — disk-pool full, bridge missing, image not found.
5. **DB constraint** — unique violation on `name` or `mac_address`; null on required field.

## Design

This plan is **diagnose-then-fix**. Phases 1–2 collect evidence; Phase 3 commits to specific changes informed by that evidence; Phase 4 verifies and deploys.

### Phase 0 — Setup
Standard `hv_status` → `hv_init` / `hv_next`. Record the execution branch in the workflow's deck.md.

### Phase 1 — Reproduce + capture

Use the deployed environment on `ubuntu-server.talkersoft.com`. The plan does NOT yet know the failure mode; this phase makes it concrete.

- Open `https://ubuntu-server.talkersoft.com/create` (or `/vms` → "Create VM" if the route moved). Pick a host + image; submit with minimal fields.
- Capture from the browser:
  - Network tab: request URL, request body, response status, response body (verbatim).
  - Console: any JS error.
  - Screenshot of the UI error toast/state.
- Capture from the server (last ~200 lines bracketing the failed request timestamp):
  - `sudo journalctl -u cloud-manager-api --since "5 minutes ago"` → save to `Retro/api.log`.
  - `sudo journalctl -u vorch_service --since "5 minutes ago"` → save to `Retro/vorch.log`.
  - `sudo journalctl -u rabbitmq-server --since "5 minutes ago"` (sanity check it's up) → save to `Retro/rmq.log` if anything interesting.
- If the error is a 5xx with no stack trace surfaced, increase API log level via env var on the systemd unit, restart, repeat.

The artifacts (request/response, screenshots, journal excerpts) live under `Retro/` of the execution workflow folder so reviewers can confirm the diagnosis.

Definition of done for this phase: a single paragraph in the workflow's `Retro/REPRODUCTION.md` that contains the verbatim error message, the HTTP status, the layer it was emitted by, and a one-line hypothesis.

### Phase 2 — Diagnose layer + identify root cause

Read the captured artifacts and walk the flow narrowing the failure layer:

| Symptom | Likely layer |
|---|---|
| 400 with validation message | web (wrong request shape) or API (DTO mismatch) |
| 404 on the create route | web route misconfig (note: `/api/v1/virtualmachine/create/{hostId}` — confirm the path didn't change) |
| 500 with `Newtonsoft.Json` or `System.NullReferenceException` | API model binding or controller logic |
| 500 with `Npgsql` | DB constraint or schema drift |
| 200 from API but VM never appears | AMQP publish failed silently OR vorch handler crashed |
| Vorch journal shows panic / libvirt error | vorch-lib `provision.InitializeHost` or infra |

Once the layer is identified, locate the offending file:line and document:
- Which repo
- File + line number
- Why it fails (one paragraph)
- The minimal change

Cross-reference against the `git log` of the relevant file to see if recent commits (especially anything in the last 30 days from PRs #11, #6, #12, #7) introduced the regression.

### Phase 3 — Implement the fix

Specific to the diagnosed layer. Open questions on the exact fix until Phase 2 completes, but the shape is:

- **If `cloud-manager-web`**: edit `CreateVmPage.tsx` or `services/api.ts`; `npx tsc --noEmit`; `npm run build`; deploy via `cloud-manager-api/scripts/web/install-web-app.py` after `sudo rm -rf .../cloud-manager-web/dist`.
- **If `cloud-manager-api`**: edit `VirtualMachineController.cs` / DTO / service; `dotnet build CloudManager.sln` clean; deploy via `cloud-manager-api/scripts/api/install-api-service.py`.
- **If `vorch-service`**: edit `handlers/vm_handlers.go` or related; `go build ./...` + `go vet ./...` clean; deploy via `sudo python3 cloud-manager-api/scripts/vorch/install-vorch-service.py`.
- **If `vorch-lib`**: edit the relevant provision file; vorch-service rebuilds against the local replace directive and is redeployed with the vorch installer above.

**Constraint**: one bug, one PR per affected repo. No opportunistic refactors. Add validation/clean error responses if absent (API) or surface real libvirt errors back through the API/UI (vorch) — but don't add silent fallback paths.

### Phase 4 — Verify + deploy

- Run the verifying build (`dotnet build` / `go build` / `tsc --noEmit`) for every repo touched.
- Apply the install scripts in dependency order: API first, then vorch, then web.
- Re-run the exact reproduction steps from Phase 1. Capture the now-successful network response + journal lines and save to `Results/RESULT.md` to prove the fix.
- Confirm `systemctl is-active` for `cloud-manager-api`, `vorch_service`, `cloud-manager-web` all return `active`.
- The VM should reach `OrchestrationStatus.Completed` in clouddb (`vm.virtual_machines.orchestration_status`) within reasonable time; verify via the database-toolkit MCP.

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000  | Setup | hv_status + hv_init/hv_next; record branch |
| 0001  | Reproduce | Trigger create-VM failure on ubuntu-server; capture request/response/journals; write `Retro/REPRODUCTION.md` |
| 0002  | Diagnose | Walk the captured evidence to a specific file:line; document the root cause |
| 0003  | Fix | Implement the minimal change in the diagnosed repo(s); typecheck/build clean |
| 0004  | Deploy + verify | Run the appropriate install-*.py; re-run reproduction; confirm success; write `Results/RESULT.md` |
| 0005  | Final | Write `Retro/LESSONS.md`; hv_ship (produces 1–N PRs depending on what was changed) |

## Constraints (carry through to execution)

- No fix speculation in Phases 0–1. Phase 2 is where commitments are made.
- One bug, one PR per affected repo. No "while we're here" cleanups.
- No disabling feature flags or removing guards as a workaround.
- Prefer surfacing real errors to the user over swallowing them.
- The plan is scoped to ONLY VM-creation. If reproduction reveals adjacent bugs, open separate plans for them.

## Open questions

These should be answered during execution (Phase 1 captures provide the evidence):

- What is the actual error message visible to the user?
- What HTTP status does the API return on the failing POST?
- Does the API log show the request hitting the controller at all (or is it failing earlier — auth, CORS, route)?
- Does `OrchestrationQueue` receive the `create-vm` message? Confirm via RabbitMQ management UI or vorch journal "Received a message" line.
- Does the vorch handler complete or panic?
- Has any VM created via the API succeeded recently (check `vm.virtual_machines.created` timestamps for the most recent rows)? If yes, what's different about the failing one?

## Repos likely in scope

The plan narrows this in Phase 2. Maximum surface area:
- `cloud-manager-web`
- `cloud-manager-api`
- `vorch-service`
- `vorch-lib`
- `workflow-exec` (workflow scaffold + Results/Lessons at ship time)

Most likely **single** repo touched: `cloud-manager-api` or `vorch-service`, based on where regressions in this flow typically land in the codebase's history.
