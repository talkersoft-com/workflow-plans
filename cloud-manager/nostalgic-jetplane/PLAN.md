# Plan: MCP + API gap-closing for VM secret-binding management

## Objective

Close the two gaps that stop an agent from fully **managing and verifying** user-defined secrets,
secret bindings, and their blueprint attachments through the cloud-manager MCP. After #1/#1.5/#2 the
surface is ~90% there, but: (1) there's no MCP tool for the existing **PATCH** on a blueprint↔binding
attachment (change `credName` / reorder), and (2) **nothing — neither API nor MCP — can read which
secret bindings were injected into a provisioned VM**, even though #2 records them in
`vm.vm_secret_bindings`. This workflow adds one API read endpoint and two MCP tools so an agent can
update an attachment and **verify what got injected into a VM** — the missing piece for an
end-to-end provision-and-verify test. This is workflow **#3** of the Secret Bindings feature. **No
DB changes, no vorch, no web.**

## Background

- **Verified in code (branch `nostalgic-jetplane` = `origin/hive`):**
  - **API** exposes: `SecretController` (list/get/create/delete), `SecretBindingController` CRUD +
    blueprint attach (`POST`, takes `credName`+`position`) / list (`GET`) / **update
    (`PATCH /api/v1/blueprint/{blueprintId}/secret-binding/{bsbId}`)** / detach (`DELETE`),
    `BlueprintController` (full), `MarketplaceController` (list + instantiate).
  - **MCP** has: `cloud_secret_*` (4), `cloud_secret_binding_*` (5 CRUD),
    `cloud_blueprint_secret_binding_attach/detach/list`, `cloud_marketplace_list/provision`, full
    blueprint/playbook tools.
- **GAP 1 (MCP-only):** the `PATCH` attachment endpoint has no MCP tool. attach/detach/list exist;
  update does not (the playbook side *does* have `cloud_blueprint_playbook_reorder`).
- **GAP 2 (API + MCP):** `#2` populates `vm.vm_secret_bindings` on provision, but nothing ever
  exposed it. An agent cannot ask "what creds were injected into this VM?" — which is exactly the
  verification step the end-to-end test needs.

## Design

### 1. API — new read endpoint (`cloud-manager-api`)
`GET /api/v1/vm/{publicId}/secret-binding` on `VirtualMachineController`. Returns the VM's
`vm_secret_bindings` usage rows:

```jsonc
[
  {
    "publicId": "vsb_…",            // the usage row
    "credName": "DBPASSWD",         // systemd credential name injected
    "secretBindingId": "sb_…",
    "secretBindingName": "postgres",
    "resolvedVaultPath": "cloudmanager/data/vm/instances/vm_…/postgres",  // a PATH, never a value
    "createdAt": "…"
  }
]
```
- Backing service method: query `vm_secret_bindings` for the VM, join `secret_bindings` for the
  name, soft-delete aware. `resolvedVaultPath` = the binding's path with `{vm_public_id}` substituted
  (templated) or the referenced `Secret.vault_path` (static) — it's a **path, not a secret**.
- **404** if the VM doesn't exist. camelCase; **public_ids only** (never a Guid).
- **The secret value is NEVER returned** — only metadata + the Vault path (same rule that keeps
  `GET /run/{id}/secret` off the MCP).

### 2. MCP — `cloud_vm_secret_binding_list` (`cloud-manager-mcp`)
Wraps the new endpoint. Input: `id` (`vm_…`). Returns the usage rows above (no value). This is the
**verification tool** for the test ("confirm `DBPASSWD` from `sb_…` was applied to this VM").

### 3. MCP — `cloud_blueprint_secret_binding_update` (`cloud-manager-mcp`)
Wraps the existing `PATCH /api/v1/blueprint/{blueprintId}/secret-binding/{bsbId}`. Inputs: `id`
(blueprint public_id), `bsbId` (`bsb_…`), optional `credName`, optional `position`. Mirrors the
shape of `cloud_blueprint_playbook_reorder` + the attach tool. Send only the provided fields.

### Conventions
camelCase; public_ids only (`PublicIdByGuidAsync`/`GuidByPublicIdAsync`); soft-delete authoritative;
secret values never in output. Mirror the existing `SecretBinding*` controller/service and the
playbook attach/reorder MCP tools — do not invent a new paradigm.

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000 | Setup | hv_status + hv_init/hv_next; record branch |
| 0001 | API — VM secret-binding read endpoint | `GET /api/v1/vm/{publicId}/secret-binding` + backing service (join, public_ids, no value, 404); unit-test the shape against a VM with a usage row |
| 0002 | MCP — `cloud_vm_secret_binding_list` | wrap the endpoint; returns usage rows; no value; verified against live API |
| 0003 | MCP — `cloud_blueprint_secret_binding_update` | wrap the existing PATCH; credName/position optional; verified against live API |
| 0004 | Verify + build + deploy | round-trip all three (attach→update→provision→list-usage) against the live API; build api+mcp; deploy api; surface the `/mcp` reload operator action; write Results + LESSONS; hv_integrate |

## Open questions

*(Scope + the no-value rule are locked. Shape resolved: metadata + Vault path only.)*

- **`resolvedVaultPath` for templated bindings** — assumes the resolved (substituted) path is the
  most useful for verification. If you'd rather see the raw `path_template` (un-substituted),
  trivial to switch. Default: resolved.
