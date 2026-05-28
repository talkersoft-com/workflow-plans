# Plan: workflow-configuration Repo — HV_HOME Source Control

## Objective

Add a `workflow-configuration` repo to the cloud-manager deck that mirrors the `~/.hv/` directory structure (`config.yaml`, `decks/`, `mcps.yaml`, `modules.yaml`). After `make install`, hive-deck writes `~/.hv/.env` with `HV_HOME` pointing at this repo. The user's shell sources this file so `HV_HOME` is always set after install. All hive-deck configuration lives in source control, completely decoupled from the hive-deck-pro binary project.

## Background

Today hive-deck configuration lives in `~/.hv/` on each machine — per-machine only, not version-controlled, not peer-reviewed, duplicated across machines. The `$HV_HOME` env var already overrides config search (priority: `$HV_HOME/.hv/` → `CWD/.hv/` → `$HOME/.hv/`). The only missing piece is a convention for which repo owns the config and a mechanism that wires `HV_HOME` to it automatically after install.

The `workflow-fragments` repo already exists in the deck for workflow YAML. `workflow-configuration` extends this pattern to the full hive-deck config: deck definitions, MCP registries, module definitions, and the main `config.yaml`. Changes to any of these go through PRs like any other code.

## Design

### New repo in cloud-manager deck

Add `talkersoft-com/workflow-configuration` under the `config` node in `cloud-manager.yaml`:

```yaml
config:
  repos:
    - talkersoft-com/workflow-configuration
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

This is a drop-in replacement for `~/.hv/` — identical structure, just in a git repo.

### HV_HOME search path resolution

Before wiring anything, audit `config.LoadSetup()` to confirm exactly how `$HV_HOME` is used. Current code searches `$HV_HOME/.hv/` (appends `.hv/`). Options:

- **Option A** — name the repo `.hv` (awkward for git)
- **Option B** — adjust search to also try `$HV_HOME/` directly when `$HV_HOME` is set, so the repo can have any name
- **Recommendation: Option B** — one extra check in `config.LoadSetup()`, repo name stays readable

### make install writes ~/.hv/.env

```makefile
install:
	CGO_ENABLED=0 go install ./cmd/hv/
	@$(MAKE) configure

configure:
	@bash scripts/configure-hv-home.sh
```

`scripts/configure-hv-home.sh`:
```bash
#!/usr/bin/env bash
# Detect a provisioned workflow-configuration repo and write HV_HOME to ~/.hv/.env
REPO_PATH="$HOME/workspace/cloud-manager/config/workflow-configuration"
ENV_FILE="$HOME/.hv/.env"
mkdir -p "$(dirname "$ENV_FILE")"
if [ -d "$REPO_PATH" ]; then
  echo "export HV_HOME=\"$REPO_PATH\"" > "$ENV_FILE"
  echo "hv: wrote HV_HOME to $ENV_FILE"
else
  echo "hv: workflow-configuration not provisioned, skipping HV_HOME"
fi
```

### Shell sourcing (one-time setup, documented)

```bash
# Add to ~/.bashrc or ~/.zshrc
[ -f ~/.hv/.env ] && source ~/.hv/.env
```

`make install` prints a reminder if the line isn't already in the shell config.

### Migration path

1. Provision `workflow-configuration` repo (via `hv init cloud-manager`)
2. Copy current `~/.hv/` contents into the repo
3. Run `make configure` to write `~/.hv/.env`
4. Source `~/.hv/.env` in shell
5. Verify `hv status cloud-manager` resolves from repo
6. Old `~/.hv/` remains as fallback for machines without `HV_HOME`

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000 | Setup | hv_status + hv_next |
| 0001 | HV_HOME path resolution | Audit `config.LoadSetup()`; adjust search to try `$HV_HOME/` directly (no `.hv/` append) when env var is set |
| 0002 | workflow-configuration repo | Create GitHub repo; seed with current `~/.hv/` contents; add to `cloud-manager.yaml` config node; provision |
| 0003 | make configure script | Add `scripts/configure-hv-home.sh` + `make configure` target; add reminder output for shell sourcing |
| 0004 | Verify + ship | Run `make configure`; source `.env`; confirm `hv status cloud-manager` uses repo; ship |

## Open questions

- **Single vs per-deck config repo**: recommendation is one shared `workflow-configuration` per user (owned by cloud-manager deck), not one per deck. Other decks reference the same config.
- **make configure vs post-install hook**: alternatively, `hv init` itself could write the `.env` when it detects a `workflow-configuration` repo was just provisioned — tighter integration, no separate make target.
