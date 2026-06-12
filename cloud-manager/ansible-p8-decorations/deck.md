# Deck: cloud-manager

## Deck name
`cloud-manager`

## Orchestration id (plan codename)
`ansible-p8-decorations` — Ansible Epic series plan 8 of 9 (executes after P4, P5, P7 are
merged).

## Repos with code changes
| Repo | What changes |
|------|-------------|
| `cloud-manager-web` | `useVarDecorations` hook, decoration/hover wiring into `AnsibleYamlEditor` props, context bar, `<SecretValue>` + masking sweep, run pages reading `rsnap` |
| `workflow-plans` | This plan folder |

**Must NOT change:** cloud-manager-api, cloud-manager-mcp, vorch-lib, vorch-service/porch, and
the P7 editor component itself — P8 plugs into C-ED extension points, it does not edit them.

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
         message: "feat: semantic decorations — secret/required badges, hover provenance, masking (P8)"
         title:   "Ansible Epic P8: metadata-driven decorations + provenance hover + masking"
         stage:   "exec"
```
