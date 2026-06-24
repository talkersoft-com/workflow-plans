# Deck: agent-arcade

## Deck name
`agent-arcade`

## Repos with code changes
| Repo | What changes |
|------|-------------|
| `talkersoft-com/agent-arcade` | The rebuild target — all implementation lands here: packaging, Go binaries, Electron main/preload/launcher, React Studio, vanilla-JS + XState + mitt Arcade. |

Reference only (NOT modified): `talkersoft-com/agent-arcade-studio` is the read-only 1:1
parity source. `talkersoft-com/agent-arcade-api` (backend) is out of scope. The
`planning/*` repos hold this plan + the execution scaffold. Repos not listed are on the
feature branch but skipped by `hv_integrate`.

## Branch
`coarse-corgi`

## Initialize (Task 0000)
```
hv_status  deck: "agent-arcade"
hv_init    deck: "agent-arcade"
```
If already provisioned on a prior feature branch with all PRs merged:
```
hv_status  deck: "agent-arcade"
hv_next    deck: "agent-arcade"
```

## Ship (per phase)
```
hv_integrate  deck: "agent-arcade"
         message: "<commit message>"
         title:   "<PR title>"
```

## Release (after parity pass, phase 0007)
```
hv_release  deck: "agent-arcade"
       title: "Release: Agent Arcade rebuild"
```
