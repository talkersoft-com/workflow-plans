# Plan: Rebuild Agent Arcade — vanilla-JS Arcade (XState + mitt) + React Studio (coarse-corgi)

## Objective

Rebuild the deprecated `agent-arcade-studio` Electron app into the empty `agent-arcade`
repo with re-architected internals, while keeping UI, assets, menus, keyboard contract, and
observable behavior **1:1**. The Arcade control surface becomes an explicit event-driven
machine (vanilla JS + XState + mitt) instead of module-level vars hand-synced to the DOM;
Studio becomes a React SPA. Go binaries, the Electron main/preload/launcher, the
`~/.hv/agent-arcade.yaml` schema, and the npm release pipeline carry over unchanged (the
pipeline gains exactly one thing: a JS bundling stage). End state: a published
`@talkersoft-com/agent-arcade` that behaves identically to the old app but no longer loses
drafts on navigation, no longer desyncs state, and reacts to Go's real events instead of
assuming success.

## Background

`agent-arcade-studio` (read-only in this deck at `agent-arcade-studio/`) is the reference
implementation. Its Arcade renderer (`arcade/renderer/renderer.js`) is a large file of
module-level `let` state (`mode`, `showTerm`, `syncMode`, `shellMode`, `recording`, `mp`,
per-agent data) hand-synced to the DOM via `draw()`/`render*()`. Three bugs recur from this:
(1) **draft loss on navigation** — input isn't durably owned per agent; (2) **state desync** —
DOM and state drift; (3) **fire-and-assume** — the UI assumes `wezterm-bridge`/`dictation-go`
commands succeeded instead of reacting to Go's events.

The two Go bridges are already the right shape to build on: `dictation-go` is a long-lived
child that emits NDJSON events (`result`/`error`/`status`/`ready`/`health_result`) keyed by
`job_id`, and the Electron main already forwards them to the renderer; `wezterm-bridge` is
exec-per-command (output + exit code, then exits — no async channel). The rebuild leans into
that asymmetry rather than papering over it.

## Design

**Paradigm boundary (non-negotiable).** The whole system is **vanilla JS** except **Studio**
(agent-setup menu + settings screens), which is **React**. The **Arcade is vanilla JS** plus
**XState** (state machines) and **mitt** (event bus) — both plain JS libraries, not UI
frameworks. No React/JSX/React-state-tooling in the Arcade. The only shared seam is the
Electron **preload/IPC contract**.

**Packaging (preserve the mechanism, extend the build).** Copy `publish.yml` + packaging from
the reference as-is in intent: release-triggered Action on `macos-latest` + `workflow_dispatch`;
version-gate (package.json version == release tag); Go build (`build:go`, `build:wezterm`, CGO +
ad-hoc codesign) + "verify Go binaries present"; WezTerm vendoring (`scripts/fetch-wezterm.sh`,
pinned tag + SHA-256); **node-pty postinstall** (`scripts/fix-node-pty.js`, restores
`spawn-helper` exec bit — **required**); `npm pack --dry-run` → `npm publish` to GitHub Packages
(`@talkersoft-com`); the `files` allowlist approach. **One sanctioned change:** add an **esbuild**
JS stage to `npm run build` (esbuild is already a devDep) that bundles Studio (React) →
`studio/dist/**` and Arcade (vanilla+XState+mitt) → `arcade/dist/**`, both added to `files`;
new order `build:js → vendor:wezterm → build:go → build:wezterm`. **Drop** the electron-builder
DMG config + `dist`/`release`/`pack` scripts — npm packaging only, **no DMG**. Package renamed
**`@talkersoft-com/agent-arcade`**; global command stays **`agent-arcade`**.

**Unchanged.** Electron main + preload + launcher (brought over as-is, edited only where a
renderer's IPC contract changes); the `~/.hv/agent-arcade.yaml` schema (agent/command
normalization, `@`-macro arg types `select`/`text`/`flag`); Go binaries + transport contracts;
three-mode structure (Launcher/Studio/Arcade); WezTerm-pane-per-agent; full keyboard contract
(`⌘⌥A` `⌘D` `⌘F` `⌘W` `⌘←/→` `^C` `Esc`).

**Arcade architecture (XState + mitt).**
- **Actor topology:** one **root Arcade machine** + a **child actor spawned per agent** (owns
  that agent's durable context: draft input, view sub-state) + a **short-lived actor per
  dictation `job_id`** that resolves only on Go's confirming event. Separate machines/regions:
  `navigation`, per-agent `dictation`, `syncMode`, and `workspaceShell`
  (`idle → opening → open → closed`; node-pty + xterm.js — the `spawn-helper` path).
- **Transitions are event-driven, never assumed; guards enforce validity.**
- **mitt boundary:** mitt = cross-component / DOM-decoupled notifications (toasts, "agent list
  changed"); XState events = machine transitions. **IPC from main (incl. Go NDJSON) is
  translated once, at the IPC seam, into XState events** for the owning machine. Components
  never reach into a machine directly.
- **Render:** reuse `arcade/renderer/index.html` + CSS **verbatim**; a thin
  `subscribe(state) → DOM` layer replaces `draw()`.

**Go integration (event-driven).** Dictation enters **`pending`** and resolves only on a Go
event (`result → confirmed`, `error → error`) — never on send. `wezterm-bridge` ops are
**optimistic-with-reconciliation**: success on exit-0, error state on non-zero exit; never block
on confirmation it cannot provide.

**Durable per-agent state.** Draft input + per-agent context live in the per-agent child actor's
XState context. Switching agents and returning preserves in-progress input (fixes the headline
"message disappears" bug).

**Recording navigation.** Setting `recordingNavBehavior = send (default) | lock`. Navigate within
the recording agent → allowed, recording continues. Navigate to another agent → **send:** commit
dictation to the **original** agent then navigate (toast "Sent to [agent]"); **lock:** block
(toast "Locked while recording"). **Esc → cancel + discard** (toast "Recording cancelled") — the
only discard path. **Async-commit:** "commit then navigate" stops recording, dispatches the
dictation job for the original agent, then navigation proceeds immediately; the per-agent/job
actor **outlives navigation** and applies `result`/`error` to the **originating** agent (not the
now-current one) — the toast is the dispatch ack; a later Go error surfaces against the
originating agent, never silently dropped.

## Implementation phases

Each phase maps to one TASK file; phases are independent and testable.

| Phase | Title | Description |
|-------|-------|-------------|
| 0000 | Setup | `hv_status` + `hv_init`/`hv_next` on the `agent-arcade` deck. |
| 0001 | Scaffold + pipeline + Go + Electron shell | New `agent-arcade` package (`@talkersoft-com/agent-arcade`, bin `agent-arcade`). Copy packaging (publish.yml, package.json **minus DMG**, `scripts/fetch-wezterm.sh`, `scripts/fix-node-pty.js`); bring over Go binaries (`go/`, `wezterm-bridge/`) and Electron `main.js`/`preload`/`launcher` as-is; add the **esbuild `build:js` stage** wired into `npm run build`. **Exit:** `agent-arcade` launches the menu-bar launcher with **stubbed** Studio + Arcade renderers; `npm pack --dry-run` includes go/bin, wezterm-bridge/bin, vendor/wezterm, studio/dist, arcade/dist. |
| 0002 | Freeze the preload / IPC contract | Document and lock the preload surface both renderers depend on, including the **Go NDJSON → renderer event** forwarding and the `wezterm-bridge` exec result shape (exit code surfaced). No renderer logic yet. **Exit:** a written IPC contract + typed preload; main forwards `dictation:event` and command results unchanged. |
| 0003 | Studio (React + Zustand) | Rewrite Studio as a React SPA with Zustand, 1:1 with the reference: agents, systems, groups, servers, dictation options, `@`-macros (incl. `select`/`text`/`flag` arg authoring). Reads/writes the **same YAML** via the existing main IPC. **Exit:** every Studio screen/action matches the reference against an identical config. |
| 0004 | Arcade foundation (machines + bus + render) | Root machine + per-agent child actors + `navigation` machine + mitt bus + the **IPC→XState event seam** + `subscribe→DOM` over the **verbatim** `index.html`/CSS. Agents rail, agent view, `⌘←/→` switching, and **durable per-agent draft** (the headline fix). **Exit:** navigate away mid-typing and back → draft intact; no DOM desync. |
| 0005 | Arcade dictation + recording-nav | `dictation` machine (per-`job_id` actor): `idle → recording → pending → confirmed \| error`, driven by Go events. `recordingNavBehavior` send/lock with guards; Esc-discard; async-commit semantics (job outlives navigation; result/error to originating agent). **Exit:** the §"Recording navigation" matrix verified end-to-end incl. a real Go round-trip. |
| 0006 | Arcade terminal surface | Live pane peek (`wezterm-bridge get-text`), send text, **sync mode** (`⌘F`), **workspace shell** (`⌘W`, node-pty + xterm.js — verify `spawn-helper`), `@`-macro picker (`select`/`text`/`flag`), `^C` interrupt — all as machines/optimistic-reconciliation. **Exit:** each surface matches the reference; non-zero `wezterm-bridge` exit shows an error state, not assumed success. |
| 0007 | Parity pass + release smoke test + ship | Per-surface parity checklist vs `agent-arcade-studio` (every menu item, every shortcut, dictation, sync, shell, macros, switching, draft preservation). Release-artifact smoke: `npm pack` → install into a temp prefix → launch `agent-arcade` → menu-bar icon + a dictation round-trip + a `⌘W` shell. Then `hv_integrate` → `hv_release`. |

## Open questions

- **Package name** `@talkersoft-com/agent-arcade` (bin `agent-arcade`) — defaulted; confirm.
- **Studio state lib** Zustand — defaulted; Redux Toolkit acceptable if preferred.
- **Phase granularity** — the Arcade is the bulk (0004–0006); if the orchestration prefers
  smaller tasks, 0006 can split (sync / shell / macros). Reviewer's call.
