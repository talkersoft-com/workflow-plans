# Plan: porch — carve the ansible orchestrator out of vorch-service, retire the Python worker

## Objective

Split the ansible-execution responsibilities out of `vorch-service` into a new sibling binary called **porch**, built in the same repo from a separate `cmd/porch/main.go` entry point. After this lands, vorch-service is libvirt-only, porch is ansible-only, and the legacy Python `cloud-manager-worker` (currently a parallel consumer on `playbook-runs`) is fully uninstalled. Go is the source of truth; no Python remains in the runtime stack.

## Background

### Why now
1. **Single-responsibility violation in vorch-service.** `vorch-service/app.go:34-80` registers, in one process: VM lifecycle consumer (`messagesub.StartConsumer`), playbook-run + cancel consumers (`StartPlaybookRunConsumer`, `StartPlaybookCancelConsumer`), collection install/remove consumers, and the ansible-runner health probe. Two unrelated domains share zero logic and want different operational shapes (libvirt is single-instance per hypervisor; ansible is long-lived + horizontally scalable).
2. **A zombie Python runner is still consuming the queue.** `/opt/cloud-manager-worker/cli/main.py` (pipx venv `/home/cloudmanager/.local/pipx/venvs/cloud-manager-worker/`, systemd unit `cloud-manager-worker.service`, running ~4 days at the time of writing) is also subscribed to `playbook-runs`. RabbitMQ has been round-robining messages between it and vorch, so ~50% of run triggers land on the Python path with its own incompatible Vault scheme (`cloudmanager/vm/instances/{vm}/{pb}` vs the new `cloudmanager/data/playbook-secrets/{runId}/...`). The README at `planning/workflow-plans/cloud-manager-ansible/README.md:40` explicitly dispositioned this worker as "delete after VORCH-PLAN Phase 1 lands" — that deletion never happened.
3. **funky-vole only works half the time.** The just-shipped per-run scoped Vault token flow (PR cloud-manager-api #16, vorch-service #9) only works when vorch wins the round-robin. With porch as the **sole** consumer, the new flow always runs.

### What's already in vorch-service (the move-list)
- `vorch-service/app.go` lines **66-78**: playbook-run + cancel consumer registration, collection install/remove consumer registration, healthprobe goroutine. **Moves to porch.**
- `vorch-service/app.go` line **80**: `messagesub.StartConsumer(conn, serverCfg)` — VM lifecycle. **Stays in vorch.**
- `vorch-service/handlers/playbook_run_handler.go`, `playbook_run_exec.go`, `playbook_cancel_handler.go`, `cancel_registry.go`, `collection_install_handler.go`, `collection_remove_handler.go`. **Move to porch.**
- `vorch-service/handlers/vm_handlers.go`. **Stays in vorch.**
- `vorch-service/messagesub/playbook_run_sub.go`, `playbook_cancel_sub.go`, `collection_install_sub.go`, `collection_remove_sub.go`. **Move to porch.**
- `vorch-service/messagesub/messagesub.go` (`StartConsumer` for VM events). **Stays in vorch.**
- `vorch-service/healthprobe/probe.go`. **Moves to porch.**
- `vorch-service/messagepub/`, `models/`, `vault/`. **Shared** — both binaries import.

### Repo gotcha — module path is `main`
`vorch-service/go.mod` declares `module main` and code imports as `"main/handlers"`, `"main/healthprobe"`, etc. This works today only because there's one binary at the package root. Adding a second `cmd/` entry point requires renaming the module to a real path (e.g. `github.com/talkersoft-com/vorch-service`) and updating all internal imports. **This is a prerequisite, not optional.** Phase 0001 in the table below.

### Constraints (carried from funky-vole — verbatim)
- camelCase JSON on the wire; public_ids only; internal UUIDs never leaked.
- Vault tokens NEVER appear in logs (`RedactingLoggerProvider` regex `hvs\.[A-Za-z0-9_\-]+` → `hvs.<REDACTED>` — already in API; porch's Go logger must redact equivalently).
- Vault tokens NEVER appear in run output streams.
- No silent fallback to long-lived cloudmanager token. Porch reads `runDTO.vaultRunToken`; if non-empty, uses it as `VAULT_TOKEN` env; otherwise falls back to vorch's existing `MintChildToken` path (unchanged).
- Revoke on terminal must be idempotent (already in API's `RevokeRunVaultAsync`).
- One feature, one PR per affected repo.

## Design

### Final repo shape (vorch-service)

```
vorch-service/
├── go.mod                       # module github.com/talkersoft-com/vorch-service
├── cmd/
│   ├── vorch-service/main.go    # libvirt only: VM lifecycle consumers + provisioning
│   └── porch/main.go            # ansible only: playbook run + cancel + collection consumers + healthprobe
├── internal/
│   ├── vm/                      # was: handlers/vm_handlers.go + libvirt provisioners
│   ├── ansible/                 # was: handlers/playbook_run_*.go, playbook_cancel_handler.go,
│   │                            #      collection_install_handler.go, collection_remove_handler.go,
│   │                            #      cancel_registry.go, healthprobe/probe.go
│   ├── ansiblesub/              # was: messagesub/playbook_run_sub.go, playbook_cancel_sub.go,
│   │                            #      collection_install_sub.go, collection_remove_sub.go
│   ├── vmsub/                   # was: messagesub/messagesub.go
│   ├── amqp/                    # shared: connection bootstrap from messagesub.go (factored out)
│   ├── pub/                     # was: messagepub/
│   ├── models/                  # was: models/
│   └── vault/                   # was: vault/
```

The boundary rule is enforced by import surface: `cmd/vorch-service` may import `internal/vm`, `internal/vmsub`, `internal/amqp`, `internal/pub`, `internal/models`, `internal/vault`. `cmd/porch` may import `internal/ansible`, `internal/ansiblesub`, `internal/amqp`, `internal/pub`, `internal/models`, `internal/vault`. **Neither cmd imports the other's `internal/{vm,ansible}` package.** This is verified by `go list -deps` in CI / acceptance.

### Wire / DB contract — unchanged
- API publishes to the same exchange/queue (`playbook-runs`) with the same DTO shape (`runDTO.vaultRunToken` included). No API endpoint changes.
- Postgres schema unchanged.
- DTOs unchanged. camelCase + public_ids preserved.
- Cancel queue handler in porch still calls the API's existing cancel endpoint (which already invokes `RevokeRunVaultAsync`).

### Assignment-tracking semantics
The Python worker did `UPDATE vm.vm_playbook_assignments SET last_run_id=..., last_run_status=..., last_applied_at=now() WHERE id=:assignment_id` at terminal (lines 317-330 of its `main.py`). **Phase 0002 must verify** whether vorch's current playbook_run_handler already writes these — if not, porch implements the same UPDATE in the same DB transaction as the terminal-status write. If the assignments UI goes stale after the Python worker is removed, that's a regression we must prevent.

### Health probe — verify API endpoint first
`vorch-service/healthprobe/probe.go` PATCHes `/api/v1/admin/feature-flags/playbooks` with `{healthy, last_probed_at}`. The hazy-kite plan (`planning/workflow-plans/cloud-manager-ansible/hazy-kite/PLAN.md`) was meant to wire the API side. **Phase 0003 must check** whether that endpoint exists and accepts that body. If not, the API change is part of this plan (a small `PATCH` action on `FeatureFlagsController`).

### Deployment
- New systemd unit `/etc/systemd/system/porch.service`, mirroring `vorch_service.service`: `Type=simple`, `User=root`, `WorkingDirectory=/kvm-automator`, `ExecStart=/kvm-automator/porch --config /kvm-automator/config-server.yaml`, `StandardOutput=journal`, `StandardError=journal`, `Restart=always`, `RestartSec=5s`, `ReadWritePaths=/kvm-automator /usr/share/ansible/collections /mnt/cloud-storage`.
- New env file `/etc/cloud-manager/porch.env`: `VAULT_ADDR`, `VAULT_TOKEN`, `VAULT_ORCHESTRATOR_TOKEN` (the existing one — porch reuses vorch's MintChildToken path when `vaultRunToken` is empty), `RABBITMQ_*`, `DBHOST/DBUSER/DBPASSWD/DBPORT/DATABASE/SSLMODE/TRUSTSERVERCERTIFICATE`, `PLAYBOOK_WORKDIR_ROOT=/kvm-automator/playbook-runs`, `PLAYBOOK_LOG_DIR=/kvm-automator/playbook-logs`, `CLOUD_MANAGER_API_URL=http://localhost:5250`.
- Vorch's existing env file drops ansible/playbook vars.
- New install scripts in `cloud-manager-api/scripts/porch/` matching the existing api/web/worker/vorch script style: `install-porch-service.py`, `start-porch-service.py`, `stop-porch-service.py`, `uninstall-porch-service.py`. (Scripts directory is in cloud-manager-api because the existing service scripts live there; we follow the precedent rather than fight it.)
- A single coordinated cutover step (documented in `Tasks/0008-TASK.md`) STOPS, DISABLES, REMOVES `cloud-manager-worker.service`, `/opt/cloud-manager-worker/`, the pipx venv, and `/etc/cloud-manager/worker.env`. Python worker leaves zero footprint.

### Cutover safety
The cutover sequence (Phase 0007) is:
1. Build + deploy porch binary, but **do not start it**.
2. Stop `vorch_service.service` (now-old, still consuming `playbook-runs`).
3. Stop + disable `cloud-manager-worker.service`.
4. Verify `rabbitmqctl list_consumers | grep playbook-runs` returns zero rows. The queue may have buffered messages — that's fine; they'll be picked up by porch in step 6.
5. Deploy the new vorch-service binary (without ansible consumers).
6. Start `porch.service`. Verify single consumer on `playbook-runs`.
7. Start `vorch_service.service` (now ansible-free).
8. Smoke test: trigger pg-on-postgres-test, observe terminal success, then remove the Python worker artifacts (Phase 0008).

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000  | Setup | hv_status, hv_init/hv_next, scaffold deck |
| 0001  | Module rename | Change `vorch-service/go.mod` from `module main` to `module github.com/talkersoft-com/vorch-service`. Update all imports (`main/handlers` → `github.com/talkersoft-com/vorch-service/internal/...` placeholders for now; the next phase moves files into those paths). Build still passes. |
| 0002  | Package carve-out | Move `app.go` to `cmd/vorch-service/main.go`. Create `cmd/porch/main.go` as an empty scaffold. Move handler/messagesub/healthprobe packages to their `internal/...` homes per the Final Repo Shape diagram. Update imports. `go build ./...` clean. **Verify** what `playbook_run_handler.go` does with assignment-tracking SQL — flag gap if absent. |
| 0003  | Porch wires consumers + healthprobe | `cmd/porch/main.go` opens an AMQP connection, registers `playbook-runs`, `playbook-run-cancel`, `collection-installs`, `collection-removes` consumers, and starts the healthprobe goroutine. **Verify** API endpoint mismatch (`PATCH /api/v1/admin/feature-flags/playbooks` vs `FeatureFlagsController` route) — if broken, add a sub-PR to cloud-manager-api fixing the route + body shape (PATCH `/api/v1/FeatureFlags/{key}/health` accepting `{healthy, lastProbedAt}`). |
| 0004  | Vorch loses ansible | `cmd/vorch-service/main.go` only registers the VM-lifecycle consumer. No imports of `internal/ansible*`. `go list -deps ./cmd/vorch-service` does not mention `internal/ansible*`; `go list -deps ./cmd/porch` does not mention `internal/vm*`. |
| 0005  | Assignment tracking + log redaction parity | If Phase 0002 found assignment tracking absent, implement the terminal-status UPDATE in `internal/ansible/handlers`. Add a Go-side `hvs\.[A-Za-z0-9_\-]+` regex log scrubber wrapping the stdlib logger so token values can never reach journal. |
| 0006  | Install scripts + systemd unit | New `cloud-manager-api/scripts/porch/*.py` mirroring vorch. New `/etc/systemd/system/porch.service` template generated by the install script. `/etc/cloud-manager/porch.env` template populated from `cloud-manager-config.env`. Build + publish the porch binary to `/kvm-automator/porch`. |
| 0007  | Cutover | Execute the 8-step cutover sequence from the Design section. After Step 6, `rabbitmqctl list_consumers \| grep playbook-runs` returns exactly one row owned by porch. |
| 0008  | Retire the Python worker | `sudo systemctl stop cloud-manager-worker && sudo systemctl disable cloud-manager-worker && sudo rm /etc/systemd/system/cloud-manager-worker.service && sudo systemctl daemon-reload && sudo rm -rf /opt/cloud-manager-worker /etc/cloud-manager/worker.env && sudo -u cloudmanager pipx uninstall cloud-manager-worker`. Verify zero footprint. |
| 0009  | End-to-end verification | Trigger pg-on-postgres-test via UI. Observe: `playbook_runs.status = 3`, `vault_run_token` cleared at terminal, secrets written under `cloudmanager/data/playbook-secrets/{runId}/postgres-test/postgres` (verified via root token). `vm_playbook_assignments` row updated. Vorch journal shows zero playbook references. Porch journal shows the full run lifecycle. Health probe round-trips. |
| 0010  | Ship | Results/RESULT.md + Retro/LESSONS.md + hv_ship |

## Acceptance criteria

1. `go build ./cmd/porch && go build ./cmd/vorch-service` both succeed from a clean checkout of vorch-service.
2. `go list -deps ./cmd/vorch-service` contains no `internal/ansible*` package. `go list -deps ./cmd/porch` contains no `internal/vm*` package.
3. After deploy: `sudo rabbitmqctl list_consumers | grep playbook-runs` shows exactly one consumer, owned by porch (`ctag-/kvm-automator/porch-1` or similar).
4. Triggering pg-on-postgres-test from the UI ends with `playbook_runs.status = 3` (Succeeded), `vault_run_token` cleared at terminal, secrets written under `cloudmanager/data/playbook-secrets/{runId}/postgres-test/postgres`.
5. `vm_playbook_assignments` row for that (vm, playbook) has `last_run_status = 3`, `last_run_id` pointing at the new run, `last_applied_at` recent.
6. `systemctl status cloud-manager-worker` → "Unit not found". `/opt/cloud-manager-worker` does not exist. `pipx list` shows no cloud-manager-worker venv.
7. Vorch-service journal shows zero references to playbook/ansible/collection after the split. Vorch only logs VM events.
8. Health probe round-trips: API's `playbooks` feature-flag row updates `last_probed_at` every probe interval; if `ansible-runner --version` fails on the host, the flag flips to `healthy=false` within one probe cycle.
9. Grepping the porch journal for `hvs\.[A-Za-z0-9]` returns zero matches across a full run lifecycle.

## Out of scope (explicitly)

- Multi-controller routing (still single-host orch deployment; deferred to a separate plan).
- Refactoring `vorch-lib` (the shared types package). Touch only if porch carve-out requires it.
- Any change to the Vault policy beyond what funky-vole already shipped.
- Splitting porch into its own git repo. Two-binaries-one-repo is the right shape until proven otherwise.
- Horizontal scaling of porch (running >1 instance). The queue semantics already allow it; just don't try it until there's a reason.

## Open questions

1. **Module path** — confirm `github.com/talkersoft-com/vorch-service` is the right import root. If the org/account name is different, swap in Phase 0001 before any imports are written.
2. **`pipx uninstall` semantics** — the Python worker venv at `/home/cloudmanager/.local/pipx/venvs/cloud-manager-worker/` may have been installed as the `cloudmanager` user; the uninstall command in Phase 0008 must be run with the right user context. Plan author may swap to a manual `rm -rf` if pipx state is broken.

## Affected repos (expected PRs)

| Repo | Scope |
|------|-------|
| `vorch-service` | Module rename, `cmd/{vorch-service,porch}` split, internal package carve-out, log redaction, assignment tracking if missing, new systemd unit template content. |
| `cloud-manager-api` | New `scripts/porch/*.py` install/start/stop/uninstall. **Conditional**: PATCH `FeatureFlagsController` to expose the health-write route if Phase 0003 verification finds it missing. |
| `workflow-exec` | This deck (charmed-panda). |

`vorch-lib`, `cloud-manager-web`, `cloud-manager-mcp`, planning/* are NOT touched.
