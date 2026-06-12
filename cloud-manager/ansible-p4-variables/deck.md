# Deck: cloud-manager

## Deck name
`cloud-manager`

## Orchestration id (plan codename)
`ansible-p4-variables` — Ansible Epic series plan 4 of 9 (executes after P1 and P2 are merged).

## Repos with code changes
| Repo | What changes |
|------|-------------|
| `cloud-manager-api` | VariableDefinition/SecretRef/VariableEvent entities (`vdef` `sref` `varev`), CHECK constraints, registry+harvest+requiredVars endpoints, `$secretRef` override validation, materialization SecretInputs expansion, edge re-key migration |
| `cloud-manager-mcp` | New `variables.ts` tools, profile registration, dist rebuild |
| `workflow-plans` | This plan folder |

**Must NOT change:** cloud-manager-web, vorch-lib, vorch-service/porch — typed secrets ride the
EXISTING SecretInputs mechanism; porch is untouched by design.

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
         message: "feat: variable registry + secret refs + required enforcement (P4 — vdef/sref/varev)"
         title:   "Ansible Epic P4: typed variables, secrets as refs, required-var enforcement"
         stage:   "exec"
```
