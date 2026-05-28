# Plan: workflow-configuration Repo — HV_HOME Source Control

## Objective

Add a `workflow-configuration` repo to the cloud-manager deck that mirrors the `~/.hv/` directory structure (`config.yaml`, `decks/`, `mcps.yaml`, `modules.yaml`). After `make install`, hive-deck writes `~/.hv/.env` with `HV_HOME` pointing at this repo. The user's shell sources this file so `HV_HOME` is always set after install. All hive-deck configuration lives in source control, completely decoupled from the hive-deck-pro binary project.

## Background

Today hive-deck configuration lives in `~/.hv/` on each machine — per-machine only, not version-controlled, not peer-reviewed, duplicated across machines. The `$HV_HOME` env var already overrides config search (priority: `$HV_HOME/.hv/` → `CWD/.hv/` → `$HOME/.hv/`). The only missing piece is a convention for which repo owns the config and a mechanism that wires `HV_HOME` to it automatically after install.

The `workflow-fragments` repo already exists in the deck for workflow YAML. `workflow-configuration` extends this pattern to the full hive-deck config: deck definitions, MCP registries, module definitions, and the main `config.yaml`. Changes to any of these go through PRs like any other code.

## Design

### New repo in cloud-manager deck

Add `talkersoft-com/workflow-configuration` to the cloud-manager deck under a dedicated `config` node (or re-use the existing `config` node that already has the `~/.hv` symlink):

```yaml
# cloud-manager.yaml
config:
  repos:
    - talkersoft-com/workflow-configuration
  symlinks: []           # remove ~/.hv symlink — repo IS the config
  show_in_workspace: true
```

The repo lives at `~/workspace/cloud-manager/config/workflow-configuration/` and contains:

```
workflow-configuration/
├── config.yaml          ← main hive-deck config (deck.root, orgs, ship, etc.)
├── decks/               ← all *.yaml deck files
│   ├── cloud-manager.yaml
│   ├── hive-deck-pro.yaml
│   └── ...
├── mcps.yaml            ← MCP server registry
└── modules.yaml         ← module definitions
```

### make install writes ~/.hv/.env

The `Makefile` install target gains a step that:
1. Looks for `~/workspace/*/config/workflow-configuration/` (or a configurable path)
2. If found, writes `~/.hv/.env`:
   ```bash
   export HV_HOME="$HOME/workspace/cloud-manager/config/workflow-configuration"
   ```
3. If not found, skips silently (safe on a fresh machine before provisioning)

### Shell sourcing

Add one line to `~/.bashrc` / `~/.zshrc` (during `make install` or documented as a one-time step):

```bash
[ -f ~/.hv/.env ] && source ~/.hv/.env
```

This is idempotent — if the file doesn't exist, nothing happens.

### HV_HOME search path

With `HV_HOME` set, `config.LoadSetup()` already looks at `$HV_HOME/.hv/` first. The `workflow-configuration` repo IS the `.hv/` directory — its root is passed as `HV_HOME` directly so the binary finds `$HV_HOME/config.yaml`, `$HV_HOME/decks/`, etc.

> Note: the existing env var search uses `$HV_HOME/.hv/` — meaning hive-deck appends `.hv/` to `HV_HOME`. The repo root either needs to be named `.hv` or the search path logic needs a small adjustment to also check `$HV_HOME/` directly (without appending `.hv/`). Resolve in Phase 0001.

### Migration

The existing `~/.hv/` contents are copied into the new repo on first provision. After that, `~/.hv/` becomes a fallback for machines that haven't set `HV_HOME` yet.

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000 | Setup | hv_status + hv_next |
| 0001 | HV_HOME path resolution | Audit `config.LoadSetup()` — determine whether `$HV_HOME` appends `.hv/` or not; adjust if needed so `HV_HOME=/path/to/workflow-configuration` works directly |
| 0002 | workflow-configuration repo | Create GitHub repo `talkersoft-com/workflow-configuration`; copy current `~/.hv/` contents into it; add to cloud-manager deck YAML |
| 0003 | make install .env write | Add install step that detects provisioned config repo and writes `~/.hv/.env`; document shell source line |
| 0004 | Provision + verify | Run `hv init cloud-manager`; confirm `HV_HOME` is set and `hv status cloud-manager` resolves from repo; ship |

## Open questions

- **HV_HOME appends .hv/ or not**: must be verified before Phase 0002. If it appends, the repo root needs to be a directory named `.hv` — awkward for a git repo. Recommend adjusting search to also try `$HV_HOME/` directly so the repo name can be anything.
- **Multi-deck config**: if two decks each have a `workflow-configuration` repo, which one wins? Recommendation: a single shared `workflow-configuration` repo per user, not per deck. The cloud-manager deck owns it but it's shared.
- **make vs install script**: `make install` is currently just `go install ./cmd/hv/`. The .env write step could instead live in a separate `make configure` target that the user runs once after provisioning.
