# Plan: Secret-binding verification fixes (MCP param guard + persisted resolved path)

## Objective

Fix the two verification-layer defects found during the live end-to-end test, so templated
secret bindings are usable **through the MCP** and the verification endpoint reports the **truth**.
The feature itself is correct — a consumer VM was injected with a source Postgres VM's actual
credential, hash-verified against Vault. These are correctness fixes only: (#2) stop the MCP
`cloud_marketplace_provision` camelCase guard from rejecting legitimate `secretBindingParams`
param-map keys, and (#3) persist + return the **actual** resolved Vault path on the
`vm_secret_bindings` usage row instead of recomputing it with the wrong VM id. This is workflow
**#5** of the Secret Bindings feature. No vorch/web changes; no change to resolution/injection.

## Background

- **Live test (branch `thriving-framehouse` = `origin/hive`):** provisioned Postgres `pg-src-1`
  (`vm_ZABTTQF68A`), then consumer `pg-consumer-1` (`vm_NVXKR24ED8`) from a templated
  `{vm_public_id}` binding, passing `secretBindingParams:[{secretBindingId, params:{vm_public_id:
  vm_ZABTTQF68A}}]`. The injected `DBPASSWD` sha256 **matched** pg-src-1's actual Vault password.
  Resolution + injection are correct; two defects remain.
- **Bug #2 (MCP):** `cloud-manager-mcp`'s `apiRequest` recursively enforces camelCase on every JSON
  key and throws on snake_case. But `secretBindingParams[].params` is a **map whose keys are
  placeholder names** (e.g. `vm_public_id`, which must match the `{vm_public_id}` placeholder) —
  user data, not wire fields. So `cloud_marketplace_provision` with `params:{vm_public_id:…}` errors
  (`snake_case key "vm_public_id" … not allowed`). Templated bindings can't be driven via MCP — the
  live test fell back to a direct API call.
- **Bug #3 (API):** `vm.vm_secret_bindings` stores only `{virtual_machine_id, secret_binding_id,
  cred_name}` — not the actual resolved path. So `GET /api/v1/vm/{publicId}/secret-binding`
  **recomputes** `resolvedVaultPath` by substituting the **listed VM's own** id into the template.
  For a cross-VM templated binding the result is wrong: the test showed
  `…/vm_NVXKR24ED8/postgres` (the consumer) instead of the true source `…/vm_ZABTTQF68A/postgres`.

## Design

### #2 — MCP camelCase guard exemption (`cloud-manager-mcp`)
Locate the recursive camelCase key check in `apiRequest` (config.ts / the request helper). Exempt
the **inner `secretBindingParams[].params` map keys** from the check — those keys are free-form
placeholder names. The guard must still apply to every structural/wire field name (including
`secretBindingId`, `params` itself, and all other request fields); only the values *inside* a
`params` object are exempt. Implement narrowly (e.g. skip recursion into the `params` object once
under a `secretBindingParams` element), not by globally relaxing the guard.

### #3 — Persist the resolved Vault path (`cloud-manager-api`)
- **Migration (one, reversible):** add nullable `resolved_vault_path` (text) to
  `vm.vm_secret_bindings`.
- **Populate at provision:** `SecretBindingResolver` already computes the concrete vault path per
  binding; thread that path through to where the `VmSecretBinding` usage row is inserted, and store
  it in `resolved_vault_path`.
- **Read endpoint:** `GET /api/v1/vm/{publicId}/secret-binding` returns the **stored**
  `resolvedVaultPath`. Fall back to the current recompute **only when the column is null** (rows
  written before this change) so old rows still return a best-effort value.
- The value is a Vault **path**, never the secret value.

### Conventions
camelCase JSON; public_ids only; soft-delete authoritative; secret values never in output; mirror
existing controller/service + MCP code paths.

### Backward compatibility
Old usage rows (`resolved_vault_path` null) → best-effort recompute (today's behavior). New rows →
the true resolved path. MCP guard change is additive (previously-valid calls stay valid).

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000 | Setup | hv_status + hv_init/hv_next; record branch |
| 0001 | API — persist + return resolved path | one reversible migration adding `resolved_vault_path` to `vm.vm_secret_bindings`; populate from the resolver at provision; read endpoint returns stored value, recompute only when null |
| 0002 | MCP — camelCase guard exemption | exempt `secretBindingParams[].params` map keys from the camelCase check; guard unchanged elsewhere |
| 0003 | E2E + build + deploy | reproduce the cross-VM flow **through MCP** `cloud_marketplace_provision` (no camelCase error); confirm `cloud_vm_secret_binding_list` returns the true source path; migration applied; build api+mcp; deploy api; `/mcp` reload note; write Results + LESSONS; hv_integrate |

## Open questions

*(Both fixes are well-scoped; the migration is additive and backward compatible.)*
- **Recompute fallback** — assumed kept for null (pre-change) rows. If you'd rather backfill old
  rows in the migration instead, note it (most are throwaway test rows, so fallback is simpler).
