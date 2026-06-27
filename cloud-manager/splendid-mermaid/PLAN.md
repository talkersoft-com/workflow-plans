# Plan: Secret Binding provisioning hardening + instantiate-time param resolution

## Objective

Fix the regression where a blueprint carrying a **templated `{vm_public_id}`** secret binding
**bricks every VM provisioned from it**, and make templated bindings actually usable by resolving
`{vm_public_id}` to an **operator-chosen source VM** (e.g. the Postgres server) at instantiate.
Two parts: **P0** makes provisioning robust — secret resolution happens *before* the VM record is
created, so a failure aborts cleanly with no orphaned VM; **P1** adds instantiate-time parameter
resolution so a consumer VM can be bound to another VM's existing secret. When done, instantiating
a "database-consumer-client" blueprint and passing the Postgres VM as the binding's source injects
that VM's creds — and a missing/unsupplied secret yields a clean 422, never a silent orphan. This
is workflow **#4** of the Secret Bindings feature. **No DB migration; vorch unchanged.**

## Background

- **Verified root cause (branch `splendid-mermaid` = `origin/hive`).** `MarketplaceController.Instantiate`:
  1. creates the VM record + fires `blueprint_instantiated` (~line 122),
  2. calls `_secretBindingResolver.ResolveForBlueprintAsync(blueprintGuid, virtualMachine.Id)` (~line 156),
  3. builds + **publishes** the `create-vm` command (~line 160+).
  In `SecretBindingResolver.ComputeVaultPath`, a templated binding does
  `PathTemplate.Replace("{vm_public_id}", vmPublicId)` with `vmPublicId` = **the new VM's own id**,
  then `ReadKvSecretAsync`, which **throws `VaultSecretNotFoundException` when the secret is absent**.
  A fresh VM has no secret at its own path → throws → caught (~227) and **re-thrown (~231)** → the
  publish (step 3) never runs. The VM is orphaned at `orchestrationStatus=0` with only `created` +
  `blueprint_instantiated`, **no create-vm command, no IP, no failure event** (e.g. `njp-e2e-1` /
  `vm_PPR04V26XF`).
- **Unaffected:** no-binding blueprints, static bindings (managed secret), and concrete-path
  templated bindings (verified: `db-consumer-1` resolved pg-test123's secret, injected, sealed, and
  recorded its `vm_secret_bindings` row). So this is not a blanket break — it's specific to the
  templated `{vm_public_id}` form, whose secret can never exist at the new VM's path.
- **Why P0 and P1 dovetail:** once the param value comes from the operator (P1), resolution no
  longer needs the new VM's id, so it can run *before* VM creation (P0) — a failure then creates
  nothing to orphan.

## Design

### P0 — Resolve-before-create + clean failure
- **Reorder `Instantiate`:** resolve the blueprint's secret bindings **before** creating the VM
  record. On any resolution failure, return **422** with the binding name + the Vault **path** that
  was missing (path only, never the value) — no VM record, no orphan.
- **Defense in depth:** if a code path still creates a VM before a later failure, record a VM
  **failure event** and mark it failed (do not leave an eventless `orch=0` orphan).

### P1 — Instantiate-time parameter resolution
- **`InstantiateRequest` gains optional `secretBindingParams`** — per-binding param values, e.g.
  ```jsonc
  "secretBindingParams": [
    { "secretBindingId": "sb_…", "params": { "vm_public_id": "vm_<sourceId>" } }
  ]
  ```
  (keyed by `secretBindingId`). This supplies the SOURCE VM for `{vm_public_id}` — the chosen
  Postgres server, not the VM being created.
- **`ResolveForBlueprintAsync` takes the supplied params.** For each templated binding, substitute
  placeholders from the supplied values. Required `source: vm` param not supplied → **422**
  validation error naming the binding; **never** fall back to the new VM's id, **never** brick.
- **MCP `cloud_marketplace_provision`** gains an optional `secretBindingParams` passthrough.

### Conventions
camelCase; public_ids only (`PublicIdByGuidAsync`/`GuidByPublicIdAsync`); soft-delete authoritative;
secret values never in output (errors carry binding name + Vault path only); mirror existing
controller/service + the playbook/marketplace MCP tools.

### Backward compatibility
No bindings → unchanged. Static binding → unchanged. Templated binding **with** supplied params →
resolves to the source VM. Templated binding **without** required params → clean 422 (not a brick).

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000 | Setup | hv_status + hv_init/hv_next; record branch |
| 0001 | API P0 — resolve-before-create + clean failure | reorder `Instantiate` to resolve bindings before VM creation; 422 on missing secret (binding name + path, no value); no orphan; failure-event guard for any post-create failure |
| 0002 | API P1 — instantiate-time param resolution | `InstantiateRequest.secretBindingParams`; `ResolveForBlueprintAsync` uses supplied params; required-param-missing → 422; templated `{vm_public_id}` resolves to the supplied source VM |
| 0003 | MCP — pass-through param | `cloud_marketplace_provision` gains optional `secretBindingParams`; round-trips to the endpoint |
| 0004 | E2E + build + deploy | provision a consumer VM bound to pg-test123 via supplied `vm_public_id` → injected/sealed; provision a templated binding with NO param → clean 422, no orphan; no-binding + static still work; build api+mcp; deploy api; write Results + LESSONS; hv_integrate |

## Open questions

- **`secretBindingParams` shape** — keyed by `secretBindingId` with a `params` map (assumed). If
  per-blueprint-attachment (`bsb_…`) keying is preferred (same binding attached twice), switch the key.
- **Missing-param policy** — assumed **hard 422** (a VM that needs a cred shouldn't boot without it).
  Confirm you don't want a "provision anyway, skip the cred" mode.
