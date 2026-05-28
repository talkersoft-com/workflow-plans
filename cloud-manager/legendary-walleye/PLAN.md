# Plan: Move workflow-configuration and workflow-fragments to hive-deck-pro

## Objective

`workflow-configuration` and `workflow-fragments` are hive-deck concerns and must live in the `hive-deck-pro` deck, not `cloud-manager`. After this change: both repos are listed in `hive-deck-pro.yaml`, cloned under `~/workspace/hive-deck-pro/config/`, removed from `cloud-manager.yaml`, and `make configure` in hive-deck-pro points `$HV_HOME` at the new path. Both decks resolve cleanly.

## Background

The previous session added `workflow-configuration` to the cloud-manager deck as a convenience. `workflow-fragments` was already there under `planning`. Neither belongs to cloud-manager — `workflow-configuration` is the authoritative hive-deck config store and `workflow-fragments` contains reusable hive-deck planning scaffolding. Having them in cloud-manager causes confusion about ownership and means `hv status cloud-manager` reports on repos that have nothing to do with cloud infrastructure.

A workaround symlink `~/workspace/cloud-manager/config/.hv → workflow-configuration` was also created during the previous session; it must be removed.

## Design

### Deck file changes

**`workflow-configuration/decks/hive-deck-pro.yaml`** — add both repos to the `config` node:

```yaml
workflow: hive-deck-pro

mcps:
  registries:
    - hive-deck-toolkit
    - protools

deck:
  repos:
    - talkersoft-com/hive-deck-pro
  config:
    repos:
      - talkersoft-com/workflow-configuration
      - talkersoft-com/workflow-fragments
    symlinks:
      - ~/.hv
    show_in_workspace: true
```

**`workflow-configuration/decks/cloud-manager.yaml`** — remove both repos:
- Remove `talkersoft-com/workflow-configuration` from `config.repos`; drop the `config` node entirely (its only repo is moving out; the symlink `~/.hv/toolkit/db` and `show_in_workspace` are no longer needed here)
- Remove `talkersoft-com/workflow-fragments` from `planning.repos`

After both changes, sync to `~/.hv/decks/` by copying the updated YAML files.

### Disk layout after the move

```
~/workspace/hive-deck-pro/
  config/
    hv                          ← existing symlink (from deck symlinks: ~/.hv)
    workflow-configuration/     ← moved from cloud-manager/config/
    workflow-fragments/         ← moved from cloud-manager/planning/

~/workspace/cloud-manager/
  config/                       ← removed entirely (empty after move)
  planning/
    workflow-fragments/         ← removed (moved to hive-deck-pro)
    workflow-exec/
    workflow-plans/
```

### configure script

`scripts/configure-hv-home.sh` in hive-deck-pro hardcodes `~/workspace/cloud-manager/config/workflow-configuration`. Change to `~/workspace/hive-deck-pro/config/workflow-configuration`. Re-run `make configure` to update `~/.hv/.env`.

## Implementation phases

| Phase | Title              | Description                                                                                      |
|-------|--------------------|--------------------------------------------------------------------------------------------------|
| 0000  | Status check       | `hv_status` both decks — all repos clean before touching anything                               |
| 0001  | Update deck files  | Edit `hive-deck-pro.yaml` and `cloud-manager.yaml` in workflow-configuration; sync to `~/.hv/decks/` |
| 0002  | Move repos on disk | `mv` both repos to hive-deck-pro workspace; remove symlink and empty dirs                        |
| 0003  | Update configure   | Fix hardcoded path in `configure-hv-home.sh`; run `make configure`; verify both decks clean     |
