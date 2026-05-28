# Plan: workflow-configuration as authoritative HV_HOME repo

## Objective

Make `workflow-configuration` the single source of truth for all hive-deck configuration. `$HV_HOME` points directly at the repo root (`~/workspace/cloud-manager/config/workflow-configuration`). Running `make configure` writes `~/.hv/.env` with that path and `hv` resolves all config from it — no `.hv/` subdirectory required, no symlinks. The `.env` file is never committed.

## Background

The previous session seeded `workflow-configuration` from `~/.hv` and added it to the cloud-manager deck, but hit a `FindHome()` issue: it strips path levels from the resolved `config.yaml` location, so when `$HV_HOME` pointed at the flat repo (no `.hv/` subdir), it landed at the wrong home. The workaround was to set `HV_HOME` to the parent `config/` dir and add a `config/.hv → workflow-configuration` symlink. That workaround needs to be replaced by a proper fix.

Additionally the migration was incomplete: `workflows/`, `workflows.yaml`, `toolkit/`, and `sync.sh` were not copied into the repo.

## Design

### Phase 0001 — Fix `FindHome()` in hive-deck-pro

`config.FindHome()` locates the hive-deck home by taking the path of the resolved `config.yaml` and stripping two path components (`.hv/<file>` → parent). When `$HV_HOME` is set and `config.yaml` lives at `$HV_HOME/config.yaml` (flat layout), stripping two levels gives the wrong directory.

Fix: when `$HV_HOME` is set, `FindHome()` returns `$HV_HOME` directly without any path arithmetic. The existing `findConfigFile` direct-path fallback already handles finding the file; `FindHome()` just needs to trust `$HV_HOME` as the home.

Also remove the direct-path fallback logic from `findConfigFile` / `findInConfigSubdir` added in the prior session — it is no longer needed since `FindHome()` will be correct and the standard `.hv/` layout within the repo is the right approach. Wait — actually check: if the repo root IS the config dir (no `.hv/` subdir), then `findConfigFile` still needs to resolve `$HV_HOME/config.yaml` directly. Keep the direct-path fallback; just fix `FindHome()`.

### Phase 0002 — Complete ~/.hv migration into workflow-configuration

Missing files/dirs to copy from `~/.hv` into the repo root:
- `workflows/` directory
- `workflows.yaml`
- `toolkit/` directory
- `sync.sh`

Add `.env` to `.gitignore` so it can never be committed.

Remove the workaround symlink `~/workspace/cloud-manager/config/.hv`.

### Phase 0003 — Update make configure + verify

Update `scripts/configure-hv-home.sh`:
- Set `HV_HOME=~/workspace/cloud-manager/config/workflow-configuration` (the repo, not the parent)
- Remove symlink creation (no longer needed)

Run `make configure`, source `~/.hv/.env`, confirm `hv status cloud-manager` resolves config cleanly.

## Implementation phases

| Phase | Title              | Description                                                                              |
|-------|--------------------|------------------------------------------------------------------------------------------|
| 0000  | Status check       | `hv_status cloud-manager` — all repos clean before starting                             |
| 0001  | Fix FindHome()     | `FindHome()` returns `$HV_HOME` directly when set; build + test pass                    |
| 0002  | Complete migration | Copy missing `~/.hv` items into repo; add `.env` to `.gitignore`; remove symlink        |
| 0003  | Configure + verify | Update configure script to use repo path; run it; `hv status` resolves config correctly |

## Constraints

- `.env` must never be committed (enforce via `.gitignore` in workflow-configuration)
- `workflow-configuration` repo has no `.hv/` subdirectory — config lives at repo root
- Symlink `~/workspace/cloud-manager/config/.hv` must be removed after the fix
- `go test ./...` must pass after Phase 0001
