# Deck: cloud-manager

## Deck name
`cloud-manager`

## Orchestration id (plan codename)
`ansible-p7-editor` — Ansible Epic series plan 7 of 9 (no data-series dependency; executes in
ledger order).

## Repos with code changes
| Repo | What changes |
|------|-------------|
| `cloud-manager-web` | `AnsibleYamlEditor` (CM6) replacing RoleFileEditor internals, Jinja2 Lezer grammar, fragment mode, diagnostics surface, theme from SCSS tokens, consumer-page swaps |
| `workflow-plans` | This plan folder |

**Must NOT change:** cloud-manager-api, cloud-manager-mcp, vorch-lib, vorch-service/porch —
web-only plan; server lint findings are consumed if present, never required.

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
         message: "feat: CodeMirror 6 Ansible YAML editor + embedded Jinja2 (P7)"
         title:   "Ansible Epic P7: real editor — CM6, Jinja2, fragments, inline lint"
         stage:   "exec"
```
