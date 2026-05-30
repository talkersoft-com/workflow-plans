# Deck: cloud-manager

## Deck name
`cloud-manager`

## Repos with code changes

| Repo | What changes |
|------|-------------|
| `vorch-service` | `internal/ansible/playbook_run_exec.go` — add `run_public_id` to extravars; change `secretsPrefix` to `cloudmanager/data/playbook-secrets/<runID>`; audit `vaultAddrOrEmpty` so `envvars` emits the configured FQDN VAULT_ADDR. Rebuild + redeploy porch systemd. |
| `cloud-manager-api` | (Possibly) extend `PATCH /api/v1/playbook/{id}` to accept `content` updates if not already wired. Add `scripts/vault/bootstrap-policies.py` (writes `playbook-run` ACL policy, updates `cloudmanager-playbook-runner` to read playbook-secrets, writes the `playbook-run` token role). Add `seed/playbooks/` directory with exported playbook YAML + `.meta.json` sidecars. |
| `cloud-manager-mcp` | Confirm `cloud_playbook_update` covers content patches; extend if not. Add `.cicd/export-playbooks.py` + `.cicd/import-playbooks.py` (idempotent). Both call the API. |
| `workflow-configuration` | `bootstrap-worker.py` — after `hv init`, call `bootstrap-policies.py` (if `VAULT_ROOT_TOKEN` is set) and `import-playbooks.py` so a fresh dev-box ends with policies, roles, and seeded playbooks ready to run. |

Repos not listed will be on the feature branch but skipped by `hv_ship`.

## Branch

`poppy-fiddle`

## Initialize (Task 0000)

The deck is already on `poppy-fiddle` from `hv_next`. Confirm:

```
hv_status  deck: "cloud-manager"
```

If somehow not on `poppy-fiddle`:

```
hv_next  deck: "cloud-manager"
```

## Ship

```
hv_ship  deck: "cloud-manager"
         message: "porch: inject run_public_id + FQDN VAULT_ADDR; seed playbooks; bootstrap vault policies; retire the six fix_*.sql triage patches"
         title:   "harden playbook execution: porch extravars, vault config, seed playbooks, bootstrap policies"
```
