# Deck: cloud-manager

## Deck name
`cloud-manager`

## Orchestration id (plan codename)
`ansible-p9-library` — Ansible Epic series plan 9 of 9, terminal (executes after P2, P5, P8 are
merged — in practice after the whole series).

## Repos with code changes
| Repo | What changes |
|------|-------------|
| `cloud-manager-web` | Library browser + graph view, composer, effective-config preview pane, used-by guardrail modals, lint-gate toggle, run-console chunk tail-follow |
| `cloud-manager-api` | Additive only: `ansible_roles` metadata fields (tags/platforms/maintainer/version) + migration + DTOs + list filters |
| `workflow-plans` | This plan folder |

**Must NOT change:** cloud-manager-mcp, vorch-lib, vorch-service/porch; no new entities or
prefixes (contracts §1 lists none for P9).

## Branch
TBD — created by Task 0000 at execution time; record the generated name here.

## Initialize (Task 0000)
```
hv_status  deck: "cloud-manager"
hv_next    deck: "cloud-manager"
```

## Ship
```
hv_ship  deck: "cloud-manager"
         message: "feat: library browser, composer, preview, guardrails, run console (P9)"
         title:   "Ansible Epic P9: library UX — browse, compose, preview, guard"
         stage:   "exec"
```
