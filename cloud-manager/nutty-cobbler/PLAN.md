# Plan: close every porch follow-up — get pg-on-postgres-test working end-to-end

## Objective

Make `pg` actually install PostgreSQL on `postgres-test` and write the generated password to Vault. Today the run reports Succeeded but does nothing because porch runs ansible-runner against implicit localhost (no inventory). Along the way: harden the porch redaction (stderr is currently unscrubbed), drop the FIX-001 `ProtectHome=no`/pipx-symlink dance by installing ansible-runner system-wide, drop the FIX-002 `--project-dir` workaround by materializing under `project/`, and clean up the abandoned Queued runs left in the DB from charmed-panda verification.

End state: trigger pg-on-postgres-test → postgres-16 installed, `app` DB created, `appuser` exists with a 32-char password, that password is also at `cloudmanager/playbook-secrets/<runId>/postgres-test/postgres` in Vault. Porch journal + log files show zero `hvs.*` matches. `systemctl cat porch.service` shows `ProtectHome=yes`. No `--project-dir` flag in porch's exec call.

## Background

Five follow-ups were captured in `planning/workflow-exec/cloud-manager/jazzy-herring/Retro/LESSONS.md`:

| # | Tag | Severity |
|---|-----|----------|
| 1 | `[[follow-up-porch-inventory-generation]]` | **Critical** — playbook does nothing without it |
| 2 | `[[follow-up-materialize-into-project-subdir]]` | Cleanup — removes a workaround |
| 3 | `[[follow-up-redactor-stderr]]` | Security — subprocess stderr bypasses the scrubber |
| 4 | `[[follow-up-ansible-runner-system-install]]` | Hardening — restore `ProtectHome=yes` |
| 5 | `[[follow-up-clean-up-stale-runs]]` | Hygiene — 7 stale Queued rows in `vm.playbook_runs` |

### Why this is one plan, not five
- Gaps 1, 2, and 4 must ship together: changing the materialize layout (#2) and adding inventory generation (#1) both touch porch's ansible-runner invocation, and the install-script change (#4) lands in the same scripts directory that gap-3's audit will touch. Splitting would mean three back-to-back PRs against `vorch-service` and `cloud-manager-api`, which is more friction than value.
- Gap 5 is a one-line SQL maintenance step; piggybacking on the rollout phase costs nothing.

### Inputs the implementer must read first
- `planning/workflow-exec/cloud-manager/jazzy-herring/Retro/FIX-002.md` — full diagnosis of the inventory gap, including what the Python worker used to do.
- `planning/workflow-exec/cloud-manager/jazzy-herring/Retro/LESSONS.md` — the five follow-up tags + their context.
- `planning/workflow-plans/cloud-manager/charmed-panda/PLAN.md` — the carve-out this builds on.
- `planning/workflow-plans/cloud-manager/nimble-orangutan/PLAN.md` — the per-run scoped Vault token model. Not changed here, but every assertion about Vault paths is verified against it.

### What's already true (verified before drafting)
- `models.CreateVmCommandMessage.VM.IPAddress` exists in `vorch-service/internal/models/command_messages.go:8`. Vorch sees the IP at VM-create time. So vorch CAN tell the API the IP — it just doesn't.
- `cloud-manager-api/src/Models/CloudManager.Entities/Models/VirtualMachine.cs` has no `IpAddress` column. Confirmed by `grep`.
- `vm.virtual_machines` Postgres table has no `ip_address` column. Confirmed by `\d`.
- ansible-runner upstream ships an apt package in Ubuntu 24.04 (`apt-cache search ansible-runner` — verify in Phase 4).

## Design

### Data model
Add `ip_address VARCHAR(45) NULL` to `vm.virtual_machines`. NULL means "unknown" (existing rows on day 1, or VMs we don't manage). One EF Core migration in `cloud-manager-api`.

`VirtualMachine` entity grows a `string? IpAddress`. DTO grows `IpAddress` mapped to wire field `ipAddress`. AutoMapper profile updated.

### Vorch → API IP write-back
When `CreateVMCommand` succeeds in vorch, it already has `message.VM.IPAddress`. Add a small HTTP call to `cloud-manager-api`:

`PATCH /api/v1/vm/{publicId}/ip-address` with body `{ "ipAddress": "10.0.150.51" }`. Controller calls a new `IVirtualMachineService.UpdateIpAddressAsync(publicId, ip)` which updates the entity and SaveChanges. 200 OK.

Vorch fires this fire-and-forget (best effort; log on failure but don't fail VM create). Same idea for `start-vm` if the IP can change on boot — we don't think it can in this stack but the call is cheap to make.

Backfill existing VMs in the install/migration phase: a one-shot Python script runs `virsh domifaddr <vmName>` for each row, parses the IPv4, and PATCHes the API. Idempotent.

### Porch — inventory generation (Gap 1)
In `vorch-service/internal/ansible/playbook_run_exec.go`, before `cmd := exec.CommandContext(...)` inside `executeOneTarget`:

1. Fetch VM details: `GET /api/v1/vm/{target.VMId}`. Add a new field `IpAddress` to `playbookRunTargetDTO` if the existing target endpoint returns it; otherwise add a small per-target fetch (one HTTP call per target). Decision: extend the target DTO server-side instead of doing a second fetch — the assignment lookup already happens in `PlaybookRunTargetService.GetByIdAsync:54`, just add the IP join.

2. Write `<workdir>/inventory/hosts`:
   ```
   <target.VMName> ansible_host=<target.IpAddress> ansible_user=cloudmanager ansible_ssh_private_key_file=env/ssh_key ansible_ssh_common_args='-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null'
   ```
   If `target.IpAddress` is empty, fail the target with a clear message: `"vm ip_address not set — vorch did not record it during VM create. backfill required."`

3. Pass `--inventory <workdir>/inventory/hosts` to the exec call. ansible-runner also auto-discovers this path so it's belt-and-braces.

### Materialize change (Gap 2)
In `cloud-manager-api/src/Services/CloudManager.Data.Services/PlaybookMaterializationService.cs:55-90`, change every `RelativePath` write so files land under `project/`:

| Before | After |
|--------|-------|
| `site.yml` | `project/site.yml` |
| `roles/{name}/{path}` | `project/roles/{name}/{path}` |

Then in porch's `playbook_run_exec.go`, drop `--project-dir <workdir> -p site.yml` and replace with `-p site.yml` only (ansible-runner now finds it under the default `<workdir>/project/site.yml`).

### Redaction hardening (Gap 3)
The current `cmd/porch/main.go` `redactingWriter` only wraps `log.SetOutput`. Subprocess stdout/stderr bypass it.

Move `redactingWriter` to its own package `internal/redact/` (was inline in `cmd/porch/main.go`) so `internal/ansible` can import and use it. Then in `playbook_run_exec.go`:

- Wrap the log file `lf` with `redact.NewWriter(lf)` before assigning to `cmd.Stderr`.
- In the stdout scan loop (`scanner.Scan()` around line 280), scrub the line before `lf.WriteString(line)`.
- Also scrub `line` before the `log_pub.publish(stdout)` call (the AMQP log streamer publishes to the UI live log view — same risk).

Net effect: every byte ansible-runner emits gets the `hvs\.[A-Za-z0-9_\-]+` → `hvs.<REDACTED>` substitution before it can land anywhere.

### ansible-runner system install (Gap 4)
Update `cloud-manager-api/scripts/porch/install-porch-service.py`:
1. Prerequisites check: try `apt install -y ansible-runner` if `which ansible-runner` returns nothing under `/usr/bin/` or `/opt/`. If apt doesn't have the package, fall back to `pipx install --global ansible-runner` (Python 3.12 pipx supports `--global` which installs to `/usr/local/share/pipx/` — accessible without `ProtectHome=no`).
2. Set `Environment="ANSIBLE_RUNNER_BIN=/usr/bin/ansible-runner"` (or `/usr/local/bin/ansible-runner` if pipx --global was used).
3. Change `ProtectHome=no` → `ProtectHome=yes` in the unit template.
4. Remove the `/usr/local/bin/ansible-runner` symlink + `/usr/local/bin/ansible-runner.real` wrapper that FIX-001 created (`uninstall-porch-service.py` also gets updated to clean these).

### Stale run cleanup (Gap 5)
One SQL in the rollout phase:
```sql
UPDATE vm.playbook_runs
SET status = 5, completed_at = now(), error_message = 'abandoned during charmed-panda verification'
WHERE status = 1 AND vault_run_token IS NULL AND created < '2026-05-29';
```
Plus the matching target row update so the UI doesn't show "Queued → no target completed":
```sql
UPDATE vm.playbook_run_targets
SET status = 5, completed_at = now()
WHERE status = 1 AND run_id IN (
  SELECT id FROM vm.playbook_runs WHERE error_message = 'abandoned during charmed-panda verification'
);
```

### Wire / DB contract changes
- New column `vm.virtual_machines.ip_address` (varchar(45), nullable).
- New endpoint `PATCH /api/v1/vm/{publicId}/ip-address`.
- Existing GET `/api/v1/vm/{id}` response grows `ipAddress` field (camelCase, optional).
- Existing GET `/api/v1/run/{runId}/target/{targetId}` response grows `ipAddress` field (sourced from the VM).
- Vorch and porch update accordingly.

No breaking changes for the web app — `ipAddress` is additive.

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000  | Setup | `hv_status`; read PLAN + LESSONS + FIX-002. |
| 0001  | Migration + entity + DTO | EF migration adding `ip_address` to `vm.virtual_machines`. Entity, DTO, AutoMapper profile updated. `dotnet build` clean. |
| 0002  | API IP-write endpoint + GET surfaces | New `IVirtualMachineService.UpdateIpAddressAsync` + controller action `PATCH /api/v1/vm/{publicId}/ip-address`. Extend GET /api/v1/vm/{id} to include `ipAddress`. Extend `PlaybookRunTargetService.GetByIdAsync` to JOIN and surface `IpAddress` on the target DTO. |
| 0003  | Materialize under project/ (Gap 2) | `PlaybookMaterializationService` rewrites every `RelativePath` to start with `project/`. Smoke-test the materialize endpoint returns the new layout. |
| 0004  | Vorch IP writeback (Gap 1, server side) | `CreateVMCommand` calls `PATCH /api/v1/vm/{id}/ip-address` after successful create. Best effort; log+continue on HTTP failure. Same call from any other place vorch knows an IP for sure. |
| 0005  | Backfill script | One-shot Python under `cloud-manager-api/scripts/porch/backfill-vm-ip-addresses.py`. Iterates `vm.virtual_machines` rows where `ip_address IS NULL`, runs `virsh domifaddr <name>`, PATCHes the API. Run during deploy (Phase 11). |
| 0006  | Porch inventory generation (Gap 1, runner side) | `executeOneTarget` writes `<workdir>/inventory/hosts` from `target.IpAddress`/`target.VMName`. Adds `--inventory` flag. Fails target cleanly if IpAddress empty. |
| 0007  | Porch drops --project-dir (Gap 2 close-out) | Once API materialize change is deployed, change exec to `ansible-runner run <workdir> -p site.yml --json`. |
| 0008  | Stderr / stdout redaction (Gap 3) | Move `redactingWriter` to `internal/redact/`. Wrap `cmd.Stderr` and the stdout scanner sink + log_pub publish call. Add a unit test that pipes `hvs.SAMPLE_TOKEN_123` through the wrapper and verifies output is `hvs.<REDACTED>`. |
| 0009  | ansible-runner system install (Gap 4) | Update `install-porch-service.py` to prefer apt > pipx --global > existing /root/.local; set `ANSIBLE_RUNNER_BIN` accordingly; flip `ProtectHome=yes`. `uninstall-porch-service.py` removes the FIX-001 symlinks. |
| 0010  | Build + deploy | Publish API; build + deploy porch (which rolls in install script changes). Apply migration. Run backfill script. Restart porch + vorch + API. |
| 0011  | Stale-run cleanup (Gap 5) | Two UPDATEs from the SQL above. Record counts before/after in CHANGE-0011.md. |
| 0012  | End-to-end verification | Trigger pg-on-postgres-test. SSH to the VM, verify postgres-16 + app DB + appuser. Read the Vault KV path with root token — fields match the password the VM was provisioned with. Porch journal + log file: zero `hvs.*`. |
| 0013  | Ship | Write Results/RESULT.md + Retro/LESSONS.md; `hv_ship`. |

## Acceptance criteria

1. **The pg playbook actually runs end-to-end against postgres-test.**
   - `playbook_runs.status = 3`.
   - `vault_run_token IS NULL` (revoked at terminal).
   - `vault_secrets_prefix = cloudmanager/data/playbook-secrets/<runId>`.
   - Vault KV at `cloudmanager/playbook-secrets/<runId>/postgres-test/postgres` has fields `host`, `port`, `db`, `user`, `password` (verified with root token).
   - `ssh cloudmanager@<postgres-test-ip>` shows postgres-16 installed; `sudo -u postgres psql -l` lists the `app` DB; `appuser` exists with that password.
2. `systemctl cat porch.service | grep ProtectHome` shows `ProtectHome=yes`.
3. `which ansible-runner` reports a path under `/usr/bin/` or `/usr/local/bin/` (pipx --global), NOT a real-file shim that follows into `/root/...`.
4. `grep -- --project-dir vorch-service/internal/ansible/playbook_run_exec.go` returns zero matches.
5. `grep -E "hvs\\.[A-Za-z0-9]" /kvm-automator/playbook-logs/<runId>.log` returns zero matches across a successful run.
6. `sudo journalctl -u porch --since "10 minutes ago" | grep -E "hvs\\.[A-Za-z0-9]"` returns zero matches.
7. `SELECT count(*) FROM vm.playbook_runs WHERE status = 1 AND created < '2026-05-29'` returns 0.
8. `SELECT count(*) FROM vm.virtual_machines WHERE ip_address IS NULL` returns 0 after the backfill script runs (assuming all known VMs are bootable; document any exceptions in CHANGE-0005).
9. `sudo rabbitmqctl list_consumers | grep playbook-runs` still shows exactly one consumer (porch) — this work doesn't regress the charmed-panda single-consumer guarantee.

## Affected repos (expected PRs)
| Repo | Scope |
|------|-------|
| `cloud-manager-api` | Migration; entity + DTO + AutoMapper; new PATCH endpoint; extend GET DTOs; materialize under `project/`; updated `scripts/porch/install-porch-service.py` + `uninstall-porch-service.py` + new `backfill-vm-ip-addresses.py`. |
| `vorch-service` | IP writeback HTTP call from `CreateVMCommand`; inventory generation in `executeOneTarget`; `redactingWriter` moved to `internal/redact/` + applied to subprocess streams; `--project-dir` flag removed. |
| `workflow-exec` | This deck (`nutty-cobbler`). |

`vorch-lib`, `cloud-manager-web`, `cloud-manager-mcp`, `infrastructure-toolbox` — NOT touched.

## Out of scope (explicitly)

- DHCP IP-change handling. Sticking with cached IP. If a VM's IP changes after its row was written, a manual re-run of the backfill script picks it up. Documented in LESSONS as `[[follow-up-vm-ip-refresh]]`.
- Per-target child Vault token minting on the runner side. The funky-vole API-side mint covers what the pg playbook needs.
- Web UI changes — `ipAddress` is additive on the wire; nothing to render in the UI right now.
- Multi-VM run semantics (parallel target execution, fan-out reporting). The pg playbook is single-target.
- Vault policy changes — the charmed-panda + nimble-orangutan setup works.

## Open questions

1. **`apt install ansible-runner` availability on Ubuntu 24.04** — Phase 4 must verify this with `apt-cache policy ansible-runner` before relying on it. If absent, fall through to `pipx install --global`. Document the chosen path in CHANGE-0009.
2. **IP source on existing VMs without a vorch-side record** — the backfill script will need `virsh domifaddr <vmName>`. Some legacy VMs might not respond (no DHCP lease, etc.). Plan: log + skip; report at end of script. Operator can populate manually via the new PATCH endpoint.
3. **`ansible_user` for postgres-test** — assumed `cloudmanager` based on standard cloud-init in this stack. Verify by SSH before Phase 6 ships; if different, parameterize via env var on porch (`PORCH_ANSIBLE_USER=cloudmanager`) so we don't hard-code.
