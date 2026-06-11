# Plan: Shared config layer + canonical path tokens (R7.3–R7.6)

## Objective

Split hv configuration into two truths: **machine truth** (each machine's `config.yaml` — its
identity, never inherited) and **shared truth** (a `deck_config:` layer holding deck definitions,
workflows, modules, profiles, rulesets). Every artifact resolves machine-first then layer,
**per file**, so one machine can override a single deck yaml or fragment without forking the rest.
All config-declared paths gain a canonical `{{WorkspaceRoot}}` token (legacy `{{DecksRoot}}` kept),
`~` expansion, and relative-to-declaring-file resolution — and mcps.yaml registries accept the same
tokens so shared registries are machine-portable. A config without `deck_config` behaves exactly
as today.

## Background

- Today HV_HOME (workflow-configuration) is a single layer: `config.yaml`, `decks/`,
  `modules.yaml`, `claude-profiles.yaml`, `gitignore-rulesets.yaml` all live together and are
  shared verbatim across machines. Machine differences have started accreting anyway
  (`profiles/<machine>/toolkit/...` for vault config; HV_PROFILE env exists since 2026-06-10) —
  but deck/workflow definitions cannot be machine-overridden at all, and machine config vs shared
  config is one undifferentiated pile.
- Path handling is inconsistent: `{{DecksRoot}}`/`{{DeckRoot}}` work in mcps.yaml args/env,
  `{{DeckRoot}}` in plan/exec folder settings, `~` only where `ExpandRoot` happens to run, and
  relative paths resolve against ambiguous bases.
- Implementation surfaces (verified): `internal/config/config.go` (configRoot/findConfigFile
  search order; LoadDeck; LoadClaudeProfiles; LoadGitignoreRulesets; modules; `Setup`),
  `internal/workflow/registry.go` (LoadRegistry — powers `name@deck` refs),
  `internal/config` fragment fallback (`LoadFragmentWithFallback`), `ops/workflow.go`
  (`configRoot`, `listWorkflowNames`), `internal/mcp/mcp.go` (`resolveArg`/`resolveEnvVal`).

## Design

### R7.3 — the layer declaration (machine config.yaml)

```yaml
# Before — one layer; this file sits beside decks/, modules.yaml, claude-profiles.yaml…
workspace:
    root: ~/workspace
    profile: default

# After — machine truth declares where shared truth lives
workspace:
    root: ~/workspace
    profile: default
deck_config: ./base        # shared layer dir; ~, {{WorkspaceRoot}}, or relative to THIS file
```

```go
// internal/config/config.go — Setup gains:
type Setup struct {
    // ...
    DeckConfig string `yaml:"deck_config"` // shared layer root; "" = single-layer (today's behavior)
}
```

The shared layer directory holds (any subset of): `decks/` (including `decks/workflows/**`),
`modules.yaml`, `gitignore-rulesets.yaml`, `claude-profiles.yaml`.

### R7.4 — per-file, machine-first resolution

One central helper; every loader goes through it:

```go
// layerPaths returns candidate paths for a config-relative artifact:
// [<machineRoot>/rel, <deckConfig>/rel] — machine first; deckConfig absent → single-entry.
func layerPaths(rel string) []string
```

- **Single-file artifacts** (`modules.yaml`, `claude-profiles.yaml`, `gitignore-rulesets.yaml`,
  one deck yaml, one workflow extension, one fragment): first existing file wins **wholesale** —
  per-file fallback, never key-level merging. Overriding one deck means copying that one yaml into
  the machine layer.
- **`config.yaml` NEVER falls back** — it is the machine identity; only the machine root is
  consulted.
- **Scans union both layers, machine wins on filename:**
  - `hv decks` listing: union of `<machine>/decks/*.yaml` and `<layer>/decks/*.yaml`
  - workflow registry (`LoadRegistry` → `name@deck` refs) and `--list`: union of both
    `decks/workflows/**` trees; a machine copy of `feature.yaml` shadows the layer copy
  - fragment lookup (`LoadFragmentWithFallback`): machine deck-fragments → layer deck-fragments →
    embedded builtin (extends today's two-step chain to three)

### R7.5 — token engine for config paths

```go
// expandPath resolves a config-declared path:
//   {{WorkspaceRoot}} → workspace.root (canonical)
//   {{DecksRoot}}     → workspace.root (legacy alias, kept indefinitely)
//   {{DeckRoot}}      → unchanged where it exists today (deck-scoped values only)
//   ~ / ~/x           → user home
//   relative          → joined to the DIRECTORY OF THE DECLARING FILE
func expandPath(declaringFile, value string, ws WorkspaceConfig) string
```

Applied to: `deck_config`, `plan_folder`, `exec_folder`, `workspace.root` (~ already), and any
future path-valued config key. Relative-to-declaring-file means `deck_config: ./base` works from
any clone location with no `../` gymnastics — but existing `../`-style values keep working.

### R7.6 — mcps.yaml gets the same tokens

```yaml
# Before — only {{DecksRoot}}/{{DeckRoot}} understood, in args/env
args: ["{{DecksRoot}}/hive-deck-pro/hive-deck-pro/mcp/dist/index.js"]

# After — same definition is machine-portable
args: ["{{WorkspaceRoot}}/hive-deck-pro/hive-deck-pro/mcp/dist/index.js"]
env:
  HV_HOME: "{{WorkspaceRoot}}/hive-deck-pro/workflow-configuration"
```

`internal/mcp/mcp.go` `resolveArg`/`resolveEnvVal` add `{{WorkspaceRoot}}` (same value as
`{{DecksRoot}}`) and `~` expansion; existing tokens unchanged.

### Machine identity selection (decision required — see Open questions)

Recommended: integrate with the existing HV_PROFILE convention —
`configRoot()` resolves the machine config as `$HV_HOME/profiles/$HV_PROFILE/config.yaml` when it
exists, else `$HV_HOME/config.yaml` (today's behavior, full back-compat). The profile config then
declares `deck_config: ../..` (or `../../base` post-migration). This gives every machine a thin,
versioned identity file in the same repo without breaking single-layer setups.

### Out of scope (follow-up, config repo not code)

Migrating workflow-configuration itself to `base/` + thin machine configs is a separate
workflow-configuration change after this code ships; this plan only makes hv capable of it.

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000 | Setup | hv_status + hv_init/hv_next |
| 0001 | Token engine | `expandPath` ({{WorkspaceRoot}} + legacy alias + ~ + relative-to-file); wire into config path keys and mcps.yaml resolveArg/resolveEnvVal; unit tests per form |
| 0002 | Shared layer resolution | `Setup.DeckConfig` + `layerPaths`; per-file fallback in every single-file loader; union scans (decks list, workflow registry, fragments three-step chain); config.yaml exempt; unit tests incl. shadowing matrix |
| 0003 | Machine identity via HV_PROFILE | Per the open-question decision: profile config selection with root fallback; docs (README section: machine vs shared truth) |
| 0004 | Verification + ship | Golden single-layer back-compat (no deck_config → byte-identical behavior incl. assembled workflow output); two-layer override matrix on a temp HV_HOME (deck shadow, fragment shadow, machine-only deck, layer-only deck); rebuild binaries + mcp dist; RESULT/LESSONS + hv_ship |

## Open questions

1. **Machine identity selection**: HV_PROFILE-based `profiles/<machine>/config.yaml` (recommended,
   above) vs requiring distinct HV_HOME per machine vs keeping root config.yaml machine-edited.
   Default if unanswered at approval: the HV_PROFILE integration.
2. **Should `claude-profiles.yaml` allow per-file fallback or stay machine-only?** Default:
   per-file fallback like the rest (it's deck-ish shared truth; machine overrides by copying).
3. Naming: `deck_config` (per R7.3) vs `shared_config`. Default: `deck_config` as specified.
