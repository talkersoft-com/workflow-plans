# Findings 0003 — Double-encoded per-VM SSH secret

Investigation deliverable for Phase 3 of `PLAN.md`. Executed 2026-06-10 in workflow `workflow-exec/cloud-manager/gloppy-snapper`. Code investigation only — no secret material was fetched from Vault, and no code was changed.

## The secret

`cloudmanager/vm/instances/<vm_public_id>/ssh` — a KV v2 secret with a single field `ssh_key` whose **value is itself a JSON string**: `{"private_key":"...","public_key":"...","created_at":"..."}` with escaped newlines. (PLAN.md listed `private_key` + `created_at`; the struct also carries `public_key`.) Every consumer must double-decode: KV envelope → `ssh_key` string → inner JSON.

## Writer (single)

**`vorch-lib/vault/client.go:47-65` — `Client.StoreSSHKey`.** It marshals the `SSHKeyData` struct (`vault/client.go:39-43`) to JSON and stores the resulting *string* as the one field `ssh_key` (lines 55-62) — that marshal-then-stringify is the double-encoding. Called from `vorch-lib/provision/provision.go:285` during VM provisioning; the companion `DeleteSSHKey` is called at `provision.go:616` on destroy.

⚠ **The writer is vorch-lib, which the deck convention forbids changing.** Any shape normalization requires a vorch-lib change and therefore cannot be done under the current convention without an explicit exception.

## Readers (three found; all locations searched are listed)

| Reader | Location | Decode behavior |
|--------|----------|-----------------|
| vorch-lib's own `GetSSHKey` | `vorch-lib/vault/client.go:68-87` | Double-decodes; errors if `ssh_key` missing. No callers found outside the file (exported library API — external consumers possible). |
| porch executor (lives in vorch-service) | `vorch-service/internal/ansible/playbook_run_exec.go:163-195` | Double-decodes `ssh_key` → inner `private_key`, **with fallbacks**: a flat `value` field, then a flat `private_key` field. Already tolerates a normalized shape. |
| `cm ssh` (cloud-manager-cli) | `internal/vaultclient/client.go:31-73`, called from `cmd/cm/main.go:46` | Double-decodes; **strict** — errors if `ssh_key` absent or inner parse fails. No fallback. |

Searched and empty: `cloud-manager-api/scripts/` contains no programmatic reader — the only hit is `scripts/vault/VAULT-CHEATSHEET.md`, operator documentation whose examples (`vault kv get -field=ssh_key ... | jq -r .private_key`) themselves encode the double-parse contract. No separate porch repo exists in the workspace (the orchestrator lives inside vorch-service); no other repo references the `vm/instances/<id>/ssh` path or `ssh_key` field.

## Options

### (a) Normalize the shape (flat fields: `private_key`, `public_key`, `created_at`)

One coordinated change: writer (`StoreSSHKey` stores the three fields directly), migration (rewrite existing `vm/instances/*/ssh` secrets; ~1 secret per provisioned VM), and readers (vorch-lib `GetSSHKey`, cloud-manager-cli `vaultclient`, cheatsheet doc; porch executor already falls back to flat `private_key` and needs no change — its `ssh_key`-blob branch becomes dead code to remove later).

Blast radius: **vorch-lib (forbidden by convention — explicit exception required)**, cloud-manager-cli, a Vault data migration over all existing VM secrets, `VAULT-CHEATSHEET.md`. Risk window: a half-migrated state breaks `cm ssh` for unmigrated VMs unless readers are made dual-shape-tolerant first (read flat, fall back to blob), which is the safe sequencing: readers-first → writer → migrate → strip fallbacks.

### (b) Document the double-encoded shape as the contract

Zero code change. The cheatsheet already documents the double-parse; add the shape to the data-model docs as canonical. Cost: every future consumer pays the double-decode tax and the `jq`-pipeline awkwardness forever; the porch executor's three-way fallback chain remains as confusion about which shape is real.

## Recommendation

**Option (a), in a separate plan** (per PLAN.md's default assumption — confirmed correct here). Rationale: there is exactly one writer, only two readers actually need changes (the third already tolerates flat fields), and the migration is small and enumerable (`vault kv list cloudmanager/vm/instances/`). The blocking factor is purely the vorch no-change convention, which makes this the wrong workflow to do it in — the new plan should carry an explicit vorch-lib exception and the readers-first sequencing above. Until then, option (b) is the de-facto state: the double-encoded shape is the contract, and this document is its record.
