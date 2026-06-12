# Deck: cloud-manager

## Deck name
`cloud-manager`

## Orchestration id (plan codename)
`ansible-p6-porch` — Ansible Epic series plan 6 of 9. **Workflow `vorch@cloud-manager`** — the
only series plan that touches vorch repos; additive-only message rules apply.

## Repos with code changes
| Repo | What changes |
|------|-------------|
| `cloud-manager-api` | Run `Mode` column, PlaybookLintResult (`plint`) + PlaybookRunLogChunk (`logc`) entities/migrations, lint + chunk endpoints, run DTO `mode` field |
| `vorch-lib` | NEW `PlaybookLintMessage` struct + queue name const (existing structs byte-identical) |
| `vorch-service` | porch: `--check` wiring, lint consumer + handler, chunk flusher, ansible-lint health probe |
| `cloud-manager-mcp` | `cloud_playbook_lint` / `cloud_playbook_check` / `cloud_run_log_chunks` in `ansible.ts`, dist rebuild |
| `workflow-plans` | This plan folder |

**Must NOT change:** cloud-manager-web; any existing vorch-lib message struct, JSON tag, queue
name, or status string.

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
         message: "feat: porch check mode, ansible-lint, chunked log streaming (P6 — plint/logc)"
         title:   "Ansible Epic P6: porch --check, lint, log chunk streaming (additive-only)"
         stage:   "exec"
```
