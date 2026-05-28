# VORCH-PLAN — Go event processor for ansible playbook runs

## Problem

`RunController.Apply` and the new multi-VM run-trigger endpoint both enqueue `{ runId }` messages to the `playbook-runs` RabbitMQ queue today, but nothing on the worker side consumes them. The existing `PLAYBOOK-INTEGRATION-PLAN.md` recommended a Python worker (`cloud-manager-worker/`) so that `ansible-runner` could be called in-process from its native language, but the team is shipping Go elsewhere (`vorch-service` already handles VM lifecycle as a Go RabbitMQ consumer). This plan resolves the worker-language question, specifies the consumer that runs ansible playbooks against VMs while materializing playbook trees from `clouddb` content (not git), and clarifies the two-token Vault strategy that keeps the per-run secret blast radius small.

## Python-vs-Go decision (resolved)

`ansible-runner` is a Python library, so a Python worker calls it in-process; a Go worker shells out to the `ansible-runner` CLI as a subprocess.

**Recommendation: Go + subprocess inside `vorch-service`.** Reasons:

- **Operational uniformity** with the existing Go RabbitMQ consumer in `vorch-service`. One service to deploy, monitor, restart, and reason about.
- **One process, one set of dependencies** — adding a separate Python service doubles the deploy surface and the runtime dependency footprint for a single use case.
- **`ansible-runner` CLI is a stable, documented surface** — `ansible-runner run <private_data_dir> --json` has been the API for years and emits structured JSON events on stdout. Treating it as a subprocess is a well-trodden pattern (AWX itself uses subprocess invocation for isolation).
- **JSON event streaming via `--json`** is already structured, so the Go side can `bufio.Scanner` lines, `json.Unmarshal`, and update `PlaybookRunTarget` rows incrementally without writing a wire-format adapter.

**Disposition for `cloud-manager-worker/`** (the empty Python scaffold at `cloud-manager-api/cloud-manager-worker/`): **delete the scaffold** once the Go consumer is in place. Add a follow-on task: remove `cloud-manager-api/cloud-manager-worker/` from the repo as part of VORCH-PLAN Phase 9.

## Queue + message contract

- **Run queue name**: `playbook-runs`. Confirm against `IPlaybookRunPublisher` implementation on the API side; if the name differs, treat the publisher's name as canonical.
- **Run message body**: minimal — `{ "runId": "run_xxxxx" }`. The worker reads everything else (playbook content, target VMs, vars, revision pin) from clouddb via the materialization API (`GET /api/v1/playbook/{id}/materialized?revision=...`) and the run-targets read API.
- **Cancel queue name**: `playbook-run-cancel` (new, introduced jointly by `API-PLAN.md` publisher and this consumer).
- **Cancel message body**: `{ "runId": "run_xxxxx" }`.
- **Shared types live in `vorch-lib/`**: `PlaybookRunMessage`, `PlaybookRunCancelMessage`, and the materialization response structs. The C# publisher's DTO names must match by JSON field name (case-insensitive on the wire). No shared codegen — Go struct + JSON tag conventions documented in `vorch-lib/`.
- **Cancel-signal mechanism: separate queue (recommended over DB poll).** The worker maintains an in-process `map[runID]context.CancelFunc`. On a cancel message, look up the func, cancel the context, which propagates to the running subprocess via `exec.CommandContext`. The subprocess receives SIGTERM, then SIGKILL after a configurable grace period (default 15s). Worker writes final `PlaybookRun.status = Cancelled`. DB-poll was considered as an alternative — rejected because it adds latency (polling interval) and load (every active worker polling for every run), and the cancel path is exactly the case where latency matters.
- **ACK strategy**: ack only after `PlaybookRun.status` reaches a terminal state (`Succeeded` / `Failed` / `Cancelled`). On crash mid-run, the message is redelivered and the worker reconciles by fetching current run status before re-executing — if already terminal, drop and ack.

## Vault token strategy (two distinct tokens per run)

The worker uses TWO different Vault tokens during a run. State this explicitly — it's easy to get wrong, and the security model depends on the distinction.

### 1. Worker's own token (long-lived, env-supplied)

`VAULT_TOKEN` env var on the `vorch-service` process. Used for:

- Reading per-VM SSH private keys at the logical path `cloudmanager/vm/instances/<vm_pub>/ssh_key` (KV v2 — the actual API call goes to `/v1/cloudmanager/data/vm/instances/<vm_pub>/ssh_key`; the `data/` segment is API-path-only, never part of the stored secret path).
- Resolving `x-secret-path` input values from the materialization manifest's `secret_inputs[]` array. The worker reads each `vault_path` with its own auth and holds the resolved map in memory for the duration of the run.
- Minting the per-run child token via `POST /v1/auth/token/create/playbook-run` (the role established by `ANSIBLE-INSTALL-PLAN.md`).

This token has broad-ish read scope (`cloudmanager/data/vm/instances/*/ssh_key`, plus any input-resolution paths reachable via the worker's policy) and the ability to mint child tokens under the `playbook-run` role. It is NOT scoped per-run — that's intentional, since the worker is trusted infrastructure, not user code.

### 2. Per-run child token (short-lived, minted per run)

Minted from the `playbook-run` token role established by `ANSIBLE-INSTALL-PLAN.md`. Scoped to per-(vm, playbook) Vault prefix via the templated policy (or the fallback broad policy, per the install plan). TTL = 1h, explicit_max_ttl = 2h from the role config. Metadata `{ vm_public_id, playbook_public_id }` set at mint time so the templated policy resolves correctly.

Used ONLY as the `VAULT_TOKEN` env var for the ansible-runner subprocess. The playbook uses it to `vault_write` its outputs under the run's predictable prefix (`cloudmanager/vm/instances/<vm_pub>/<pb_pub>/...`). The token cannot read VM SSH keys, cannot read other VMs' outputs, cannot read input-resolution paths — it can only write under its own scoped prefix.

Revoked when the run finishes (success, failure, or cancel) via `POST /v1/auth/token/revoke`.

### Where the orchestrator stopped minting

In the original `PLAYBOOK-INTEGRATION-PLAN.md` (Worker Design section, now superseded), the orchestrator minted the child token and handed it to a Python worker. In this design, the worker mints its own child token from its own authority — because the worker is the thing that calls ansible-runner and is the right place to mint and revoke ephemerally. The existing `RunController.GetSecret` endpoint (which reads published secrets for the UI) keeps using the API's own `VAULT_TOKEN` env var and is unchanged.

## Worker Vault auth method

- **Recommended for v1**: static service token via `VAULT_TOKEN` env var. Matches the existing cloud-manager-api pattern; minimal operational surface; ops team already knows how to provision and rotate service tokens.
- **Future migration**: AppRole auth so the worker can rotate its own credentials without operator intervention. Documented as a follow-on plan, not a v1 blocker.

## Per-message flow (numbered)

For each `{ runId }` message pulled from `playbook-runs`:

1. Pull `runId` from message envelope.
2. Fetch run via API (`GET /api/v1/run/{runId}`). If `status` is already terminal, ack and drop (idempotent crash recovery).
3. Mark `PlaybookRun.status = Running, started_at = now()` via API.
4. Resolve the run's playbook + revision via the materialization API. Receive `files[]` + `secret_inputs[]` + `playbook_public_id` + `playbook_revision_id`.
5. Create a tmpfs work directory at `${PLAYBOOK_WORKDIR_ROOT}/<runId>/`. Materialize the manifest to disk: for each `{ relative_path, content }` in `files[]`, mkdir-p the parents and write the file with 0600 perms.
6. Resolve secrets ONCE per run (they're per-playbook, not per-target): for each entry in `secret_inputs[]`, use the worker's own token to read the value from Vault. Hold the resolved map in memory. Mark any tasks consuming these in the playbook with `no_log: true` (this is a playbook-author responsibility documented in `meta/argument_specs.yml` notes — the worker can't enforce it).
7. Fetch the list of `PlaybookRunTarget` rows for this run via `GET /api/v1/run/{runId}/target`. For each target, in order:
   a. Pull per-VM SSH private key from Vault (worker's own token) at `cloudmanager/vm/instances/<vm_pub>/ssh_key`. Write to `<workdir>/env/ssh_key` with 0600 perms.
   b. Mint a per-run child token from the `playbook-run` role (worker's own token authority). TTL 1h, metadata `{ vm_public_id, playbook_public_id }`.
   c. Build `<workdir>/env/extravars` JSON = `{ secrets_prefix, vm_public_id, playbook_public_id, ...vars_override_snapshot, ...resolved_secret_inputs }`. The resolved secrets are merged in here as ordinary extra-vars; playbook tasks consuming them should be marked `no_log: true` so they don't appear in stdout.
   d. Build `<workdir>/env/envvars` JSON = `{ VAULT_ADDR, VAULT_TOKEN: <child token> }`. The child token is what ansible-runner sees, NOT the worker's own token.
   e. Mark `PlaybookRunTarget.status = Running, started_at = now()` via the targets PATCH endpoint (internal — same controller, write-side).
   f. Exec `ansible-runner run <workdir> --json` as a subprocess via `exec.CommandContext(runCtx, ...)`. Stream stdout line-by-line; `json.Unmarshal` each line into an event struct. Append events to the run log file at `${PLAYBOOK_LOG_DIR}/<runId>.log` and update `PlaybookRunTarget.{status, stats, output}` periodically (every N events or N seconds, whichever first — default 25 events / 2s).
   g. Between targets (and after each subprocess exit), check `PlaybookRun.status`: if `Cancelled` or our context was cancelled, break the loop and mark remaining targets as `Cancelled` without executing them.
   h. On subprocess exit: list `cloudmanager/vm/instances/<vm_pub>/<pb_pub>/` in Vault (using the worker's own token; the child token has list rights too but it's about to be revoked), write the result into `PlaybookRun.secrets_published` JSON.
   i. Revoke the child token via `POST /v1/auth/token/revoke`.
8. Aggregate per-target statuses into `PlaybookRun.status`:
   - All Succeeded → `Succeeded`
   - Any Failed → `Failed`
   - Any Cancelled and none Failed → `Cancelled`
   - Mixed Succeeded/Failed → `Failed`
9. Set `completed_at`, write final `stats`, and update `VmPlaybookAssignment.{ last_run_id, last_run_status, last_applied_at }` for each target whose `assignment_id` is non-null (ad-hoc targets with null `assignment_id` are skipped).
10. Shred (best-effort `shred -u` on the SSH key file) and `rm -rf` the workdir.
11. Ack the message.

If steps 3–10 fail with an unrecoverable error (DB unreachable after retries, etc.), do NOT ack — let RabbitMQ redeliver. The reconcile in step 2 prevents double-execution.

## Health probe

Goroutine on `vorch-service` startup, ticks every N minutes (default 5).

Each tick:

1. `exec.Command("ansible-runner", "--version")` — verifies the binary is installed and runnable.
2. If exit 0: `PATCH /api/v1/admin/feature-flags/playbooks` (or the exact endpoint shape used by `FeatureFlagsController` — confirm against the controller) with `{ healthy: true, last_probed_at: now() }`.
3. If exit non-zero: `PATCH` with `{ healthy: false, last_probed_at: now() }` and log the error.

Also probed at boot so a restart re-evaluates immediately (vs. waiting up to 5 min for the first tick).

## Shared types in vorch-lib

Lives under `vorch-lib/` so both the existing VM-lifecycle handlers and this new run handler can consume the same wire formats. Adds:

- `PlaybookRunMessage { RunID string \`json:"runId"\` }` — body of the `playbook-runs` queue message.
- `PlaybookRunCancelMessage { RunID string \`json:"runId"\` }` — body of the `playbook-run-cancel` queue message.
- `MaterializedPlaybook { PlaybookPublicID string, PlaybookRevisionID string, Files []MaterializedFile, SecretInputs []SecretInputRef }` — response from `GET /api/v1/playbook/{id}/materialized`.
- `MaterializedFile { RelativePath string, Content string }`.
- `SecretInputRef { ExtravarName string, VaultPath string }`.
- `PlaybookRunStatus` string constants matching the API enum string values exactly (`"queued"`, `"running"`, `"succeeded"`, `"failed"`, `"cancelled"`).

The C# publisher's DTOs (`PlaybookRunMessage` etc.) must use field names that match by JSON tag, case-insensitively. The Go test suite for `vorch-lib` includes a round-trip test that unmarshals known-good JSON the API produces.

## Configuration

Match existing `vorch-service/config/` conventions (env-driven with optional config file overlay). New settings:

| Variable | Default | Purpose |
|---|---|---|
| `ANSIBLE_RUNNER_BIN` | `ansible-runner` | Subprocess binary to invoke (allows pinning a pipx path). |
| `PLAYBOOK_WORKDIR_ROOT` | `/var/lib/cloud-manager/playbook-runs` | tmpfs-preferred parent dir for per-run workdirs. |
| `PLAYBOOK_LOG_DIR` | `/var/log/cloud-manager/playbook-runs` | Where per-run log files go. |
| `VAULT_ADDR` | (already in use elsewhere) | Vault API endpoint. |
| `VAULT_TOKEN` | (env-supplied, v1) | Worker's own service token. |
| `CLOUD_MANAGER_API_URL` | (already in use elsewhere) | Base URL for the API the worker fetches from. |
| `PLAYBOOK_HEALTH_PROBE_INTERVAL` | `5m` | Tick rate for the ansible-runner version check. |
| `PLAYBOOK_RUN_TIMEOUT` | `2h` | Max subprocess wall-clock per run; kill on timeout. Matches Vault token explicit_max_ttl. |
| `PLAYBOOK_CANCEL_GRACE` | `15s` | SIGTERM → SIGKILL grace on cancel. |

## Failure modes + recovery

- **Subprocess hangs** → killed after `PLAYBOOK_RUN_TIMEOUT` (default 2h, matches Vault token ceiling). Final status = `Failed`. Child token revoked. Workdir cleaned.
- **API unreachable mid-run** → retry with exponential backoff (3 attempts, 1s/4s/16s). If still failing, log and leave the message unacked so RabbitMQ redelivers. Subsequent reconcile in step 2 picks up where we left off (or drops if status went terminal by some other path).
- **Worker crash mid-run** → message redelivered. Step 2 reconciles: if status is already terminal (e.g. operator cancelled while we were dead), ack and drop; otherwise re-execute idempotently (`PlaybookRunTarget` rows get overwritten — note that this loses partial output if a target was mid-execution, accepted v1 trade-off).
- **Disk full at materialize step** → fail fast, status = `Failed` with `error_message = "workdir creation failed: ..."`. No partial workdir left behind.
- **Vault unreachable for SSH key or input resolution** → mark target as `Failed` with explicit error; continue to next target if multi-target. The run as a whole is `Failed`.
- **Vault unreachable for child-token mint** → same: mark target as `Failed`, continue.
- **ansible-runner missing** → caught by the health probe before any runs land here; flag flips to `healthy=false` and the API rejects new runs with 503 (per the existing feature-flag three-state behavior in `PLAYBOOK-INTEGRATION-PLAN.md`).

## Phases

1. **Bare consumer**: subscribe to `playbook-runs`, fetch run, log message, ack. No execution.
2. **Materialize playbook** to disk from the materialization API; no subprocess yet.
3. **Run ansible-runner against a single target** with a hard-coded SSH key path (no Vault yet). Verifies the subprocess + JSON event parsing + status writebacks work end-to-end.
4. **Vault integration phase 1**: worker's own token reads SSH key + resolves `secret_inputs[]`. Subprocess still gets a non-scoped token.
5. **Per-run child token**: mint from `playbook-run` role, hand to ansible-runner subprocess as `VAULT_TOKEN`. Revoke on completion. Verify outputs land under the predictable Vault prefix.
6. **Multi-target loop** + per-target status updates. Worker now handles the multi-VM run shape from `API-PLAN.md`.
7. **Cancel subscription** + context-cancellation + SIGTERM/SIGKILL + final status aggregation. Joint ship with API-PLAN's cancel endpoint.
8. **Health probe** + `feature_flags.playbooks.healthy` writes.
9. **Delete `cloud-manager-api/cloud-manager-worker/`** Python scaffold. The Go consumer is canonical.
