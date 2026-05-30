# Plan: harden the postgres playbook execution path

## Plan overview

This plan replaces six in-DB SQL patches we used to triage the `pg-14-jammy` postgres playbook with real code fixes across porch, cloud-manager-api, and the playbook content itself. Porch will inject `run_public_id` and use the FQDN Vault URL so playbooks can use the existing per-run-scoped Vault policy without `delegate_to: localhost`, `validate_certs: false`, or `python3-hvac` on every VM. A new export/import pair gives the playbook bodies a git source-of-truth, and a `bootstrap-policies.py` script provisions the Vault `playbook-run` policy + role automatically. When complete, a fresh dev-box bootstrap ends with a green playbook run end-to-end.

## Objective

End-to-end success for `cloud_playbook_run` against a freshly-provisioned Jammy VM with no manual SQL, no hand-edited Vault policies, and no per-VM `hvac` install. Playbook writes the generated postgres password to `cloudmanager/data/playbook-secrets/<runId>/<host>/postgres` using the per-run-scoped Vault token cloud-manager-api already mints. The cloud-manager-web Run detail page surfaces the password via the orchestrator policy's read path. A dev-box rebuilt via `bootstrap-worker.py` reaches the same working state without operator intervention.

## Background

While troubleshooting the postgres playbook this session, we hit a chain of failures and worked around each in `vm.playbooks.content` via direct `psql` UPDATEs (six `fix_*.sql` files total). The final-state playbook works on the dev box but masks four real defects:

1. **`pg_run_id` undefined** — porch's `vorch-service/internal/ansible/playbook_run_exec.go` (line ~222) builds the extravars map with `secrets_prefix` (wrong path: `cloudmanager/vm/instances/...` instead of `cloudmanager/data/playbook-secrets/<runId>`), `vm_public_id`, and `playbook_public_id`. No `run_public_id`. Worked around by SQL-patching the playbook to use `{{ vm_public_id }}`, which broke the per-run policy cloud-manager-api creates (`playbook-run-<runId>` → grants only `cloudmanager/data/playbook-secrets/<runId>/*`).
2. **VAULT_ADDR=https://127.0.0.1:8200** — porch sets this in `env/envvars`. Unreachable from target VM, TLS cert is for `ubuntu-server.talkersoft.com`. Worked around with `validate_certs: false` + `delegate_to: localhost`. Required also adding `become: false` and `ansible_remote_tmp: /tmp/.ansible-porch-tmp` because porch's systemd unit has `ProtectHome=yes` (blocks `/root/.ansible/tmp`).
3. **Playbook content lives only in `vm.playbooks.content`** — no UI editor surfaced; no git source-of-truth; all six fixes applied via direct SQL with manual revision rows.
4. **Vault `playbook-run` policy + role bootstrap done by hand** — `vault policy write playbook-run` + `vault write auth/token/roles/playbook-run allowed_policies="playbook-run" ...` were typed by the operator into a terminal. Nothing in any repo provisions them. The orchestrator policy `cloudmanager-playbook-runner` also reads the wrong prefix (`cloudmanager/data/vm/instances/+/+/*` instead of `cloudmanager/data/playbook-secrets/+/*`), so the UI can't surface what runs wrote.

## Design

### Phase 1 — Porch extravars + secrets_prefix

`vorch-service/vorch-service/internal/ansible/playbook_run_exec.go`:

```go
// Before
secretsPrefix := fmt.Sprintf("cloudmanager/vm/instances/%s/%s", target.VMId, mat.PlaybookPublicID)
extravars := map[string]interface{}{
    "secrets_prefix":     secretsPrefix,
    "vm_public_id":       target.VMId,
    "playbook_public_id": mat.PlaybookPublicID,
}

// After
secretsPrefix := fmt.Sprintf("cloudmanager/data/playbook-secrets/%s", runID)
extravars := map[string]interface{}{
    "secrets_prefix":     secretsPrefix,
    "vm_public_id":       target.VMId,
    "playbook_public_id": mat.PlaybookPublicID,
    "run_public_id":      runID,           // NEW
}
```

`runID` is already the run public_id string (`run_XXX`) — porch uses it as the workdir path `/kvm-automator/playbook-runs/<runID>/`.

Build via `vorch-service/.cicd/build-vorch.py --target porch` (create the script if missing), deploy via `vorch-service/.cicd/deploy-vorch.py` (or hand-deploy in this phase with `sudo systemctl restart porch`).

### Phase 2 — VAULT_ADDR resolution

Audit `vorch-service/vorch-service/internal/ansible/playbook_run_exec.go` `vaultAddrOrEmpty(h.Vault)`. The current `envvars` JSON emits `https://127.0.0.1:8200`; should emit the configured `https://ubuntu-server.talkersoft.com:8200` from `/kvm-automator/config-server.yaml` `vault.address`.

```yaml
# Before (env/envvars JSON)
{"VAULT_ADDR": "https://127.0.0.1:8200", ...}

# After
{"VAULT_ADDR": "https://ubuntu-server.talkersoft.com:8200", ...}
```

The TLS cert validates cleanly against the FQDN, so `validate_certs: false` is unnecessary once this lands.

### Phase 3 — Playbook revert to per-run scoping

Edit `pg-14-jammy` and `pg-16-noble` via `cloud_playbook_update` (NOT raw SQL). For each:

```yaml
# Before
vault_secret_path: "cloudmanager/data/playbook-secrets/{{ vm_public_id }}/{{ inventory_hostname }}/postgres"
# After
vault_secret_path: "cloudmanager/data/playbook-secrets/{{ run_public_id }}/{{ inventory_hostname }}/postgres"
```

```yaml
# Before (vault_kv2_write task)
community.hashi_vault.vault_kv2_write:
  url: "{{ vault_addr }}"
  auth_method: token
  token: "{{ vault_token }}"
  validate_certs: false
  engine_mount_point: cloudmanager
  path: "playbook-secrets/{{ vm_public_id }}/{{ inventory_hostname }}/postgres"
  data: ...
delegate_to: localhost
become: false
vars:
  ansible_remote_tmp: /tmp/.ansible-porch-tmp

# After
community.hashi_vault.vault_kv2_write:
  url: "{{ vault_addr }}"
  auth_method: token
  token: "{{ vault_token }}"
  engine_mount_point: cloudmanager
  path: "playbook-secrets/{{ run_public_id }}/{{ inventory_hostname }}/postgres"
  data: ...
```

```yaml
# Before (apt install)
- "postgresql-{{ pg_version }}"
- python3-psycopg2
- python3-hvac
- acl

# After
- "postgresql-{{ pg_version }}"
- python3-psycopg2
- acl
```

If `cloud_playbook_update` MCP tool doesn't yet accept `content`, extend the API `PATCH /api/v1/playbook/{id}` and the MCP tool signature first as a sub-phase.

### Phase 4 — Playbook source-of-truth

Add `cloud-manager-mcp/.cicd/export-playbooks.py`:

```
GET /api/v1/playbook/list  →  for each: GET /api/v1/playbook/{id}
  → write seed/playbooks/<name>.yaml   (just the content)
  → write seed/playbooks/<name>.meta.json (description, argumentSchema, outputSchema, current revision public_id)
```

Add `cloud-manager-mcp/.cicd/import-playbooks.py`:

```
walk seed/playbooks/*.yaml
  for each: GET /api/v1/playbook/list   → find by name
    if missing → POST /api/v1/playbook
    if present and content differs → PATCH /api/v1/playbook/{id}
    if present and same → no-op
```

Run `export-playbooks.py` once now and commit the resulting `cloud-manager-api/seed/playbooks/` directory. That becomes the git source-of-truth.

### Phase 5 — Vault policy + role bootstrap script

`cloud-manager-api/scripts/vault/bootstrap-policies.py` (Python, takes `--vault-addr` and `--root-token`):

```python
# Create or update three policies + one token role:
#
# 1. playbook-run  (attached to short-lived child tokens)
#      path "cloudmanager/data/playbook-secrets/*"     { capabilities = ["create","update","read"] }
#      path "cloudmanager/metadata/playbook-secrets/*" { capabilities = ["read","list"] }
#
# 2. cloudmanager-playbook-runner  (UPDATED — orchestrator policy)
#      ... existing token create/revoke paths preserved ...
#      path "cloudmanager/data/playbook-secrets/+/*"     { capabilities = ["read"] }
#      path "cloudmanager/metadata/playbook-secrets/+/*" { capabilities = ["list","read"] }
#
# 3. token role: playbook-run
#      allowed_policies = ["playbook-run"]
#      token_explicit_max_ttl = "2h"
#      orphan = true
#      renewable = false
```

Document in `cloud-manager-api/README.md` and invoke from `install-porch-service.py` (or a new `bootstrap-vault.py`) so a fresh deploy lands a working Vault config without manual `vault` CLI calls.

### Phase 6 — Bootstrap-worker integration

`workflow-configuration/bootstrap-worker.py` gains two steps after `hv init`:

1. Call `cloud-manager-api/scripts/vault/bootstrap-policies.py` (if a `VAULT_ROOT_TOKEN` env var is provided).
2. Call `cloud-manager-mcp/.cicd/import-playbooks.py` against the freshly-provisioned API.

So a `python3 bootstrap-server.py` end-to-end leaves the dev-box with policies, roles, AND seeded playbooks ready to run.

### Phase 7 — E2E verify + Retro/LESSONS

`cloud_playbook_run` against `vm_GY68JPPKDN` (or a fresh Jammy VM) must:

- Succeed (status=3) with no manual SQL or Vault CLI ops.
- Write to `cloudmanager/data/playbook-secrets/<runId>/<host>/postgres`.
- Read back through `cloud_run_get` showing the Vault path in the surfaced output.

Then a separate teardown + rebootstrap test:

- `cloud_vm_force_delete` (or its successor — the soft-delete + scheduled cleanup) the test VM.
- Wipe + re-bootstrap via `bootstrap-server.py`.
- Run the same playbook against a freshly-provisioned VM. Expect green without operator intervention.

`Retro/LESSONS.md` enumerates the six triage SQL patches and which phase replaced each.

## Implementation phases

| Phase | Title                              | Description |
|-------|------------------------------------|-------------|
| 0000  | Setup                              | hv_status; confirm execution branch across all repos |
| 0001  | Porch extravars + secrets_prefix   | Patch playbook_run_exec.go to inject run_public_id and fix the prefix; rebuild + restart porch |
| 0002  | VAULT_ADDR FQDN                    | Audit vaultAddrOrEmpty path; emit the configured FQDN; verify with a sample run's env/envvars |
| 0003  | Extend cloud_playbook_update (if needed) | API PATCH + MCP tool accept `content`; build + restart api + rebuild mcp dist |
| 0004  | Revert playbook content            | Use cloud_playbook_update to roll back the six triage workarounds in pg-14-jammy + pg-16-noble |
| 0005  | Vault bootstrap script             | Add scripts/vault/bootstrap-policies.py; run it against the dev box; verify policies + role |
| 0006  | Playbook export + import scripts   | Add .cicd/export-playbooks.py + import-playbooks.py; commit seed/playbooks/ |
| 0007  | Bootstrap-worker integration       | Wire policy + import calls into bootstrap-worker.py |
| 0008  | E2E verify                         | Fresh playbook run, end-to-end check from API → porch → Vault → UI surface |
| 0009  | Teardown + rebootstrap verify      | Full bootstrap-server.py cycle ends with a green playbook run, no operator intervention |
| 0010  | Write Results/RESULT.md + Retro/LESSONS.md; hv_ship |

## Open questions

- **Should `cloud_playbook_update` MCP tool also accept argument_schema / output_schema patches**, or stay content-only? Current proposal: content only in this PR; schemas in a follow-up.
- **`bootstrap-policies.py` invocation**: silent skip if `VAULT_ROOT_TOKEN` isn't set, or hard fail? Proposal: skip with a printed warning, so a CI run without the env doesn't break.
- **Should `import-playbooks.py` allow new playbooks in the DB that aren't in the seed directory** (i.e. operator-authored), or treat any DB drift as something to overwrite? Proposal: preserve DB-only playbooks, only sync the ones in the seed directory.
