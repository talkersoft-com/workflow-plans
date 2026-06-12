# Plan: Ansible inventories as records — inventories, groups, hosts, host/group vars (ansible-p1-inventory)

> Plan 1 of 9 in the Ansible Epic series (windward-steamboat). Conforms to
> `master-plans/ANSIBLE-EPIC-CONTRACTS.md`. Workflow: `feature@cloud-manager`.
> Satisfies epic §2 (partial: inventories/host vars/group vars as records); P5's resolution
> engine depends on these records. Exposes contract **C-INV**. Depends on: none.

## Objective

Make Ansible inventories first-class database records: named inventories containing groups
(nestable), hosts (usually backed by a `vm_` record), group membership, and host/group vars as
individually addressable rows. Today a "run target" is just a `PlaybookRunTarget` pointing at a
VM with one opaque `varsOverride` jsonb; nothing models groups, group vars, or host vars, so
precedence-aware resolution (P5) has nothing to resolve against. This plan ships the entities,
APIs, events, MCP tools, and the epic's feature flag — no behavior change to existing run flows.

## Background

- Current state on main: runs target VMs through `VmPlaybookAssignment` (`asgn`) and
  `PlaybookRunTarget` (`prt`); porch generates a one-line inventory file per target VM
  (`playbook_run_exec.go`). There is no inventory entity anywhere.
- Conventions (binding, from contracts §11): Audit base, `EntityPrefixRegistry` public ids,
  camelCase wire / snake_case columns, append-only event tables, feature-flag gating.
- The `ansible` Postgres schema already exists (collections); new entities land there.
- This plan creates the epic-wide feature flag `ansible-studio` (contracts §11 decision) that
  P2–P9 reuse.

## Design

### Data model (schema `ansible`, contracts §1)

| Entity | Prefix | Notes |
|---|---|---|
| Inventory | `inv` | Name (unique where deleted_at is null), Description; soft-delete like `Blueprint` |
| InventoryGroup | `invg` | FK Inventory (Cascade), Name, ParentGroupId self-FK nullable (Restrict); unique (inventory_id, name) |
| InventoryHost | `invh` | FK Inventory (Cascade); VirtualMachineId FK nullable (Restrict); HostName, AnsibleHost (address) for external hosts; unique (inventory_id, host_name) |
| InventoryGroupMembership | `igm` | FK group + host (Cascade); unique pair |
| GroupVar | `gvar` | FK InventoryGroup (Cascade); Name, Value jsonb; unique (group_id, name) |
| HostVar | `hvar` | FK InventoryHost (Cascade); Name, Value jsonb; unique (host_id, name) |
| InventoryEvent | `invev` | append-only, VmEvent pattern; event types per contracts §6 |

Var values are untyped jsonb in this plan; P4 retrofits typing/sensitivity by linking vars to
`VariableDefinition` records — columns here are designed so that retrofit is additive
(nullable `variable_definition_id` reserved in P4's migration, not this one).

Group nesting: ParentGroupId must stay acyclic — service-level cycle check on create/update
(400 on cycle), same approach P2 later generalizes in the edge graph.

### API (cloud-manager-api, contracts §3)

All routes `[RequireFeatureFlag("ansible-studio")]`:

- `api/v1/inventory` — GET list, GET `/{id}`, POST, PATCH `/{id}`, DELETE `/{id}` (soft)
- `api/v1/inventory/{id}/group` — GET, POST; PATCH/DELETE `/{gid}`
- `api/v1/inventory/{id}/group/{gid}/var` — GET, PUT (upsert by name), DELETE `/{name}`
- `api/v1/inventory/{id}/group/{gid}/member` — GET, POST (hostId), DELETE `/{invhId}`
- `api/v1/inventory/{id}/host` — GET, POST (vmId or external hostName+ansibleHost); PATCH/DELETE `/{hid}`
- `api/v1/inventory/{id}/host/{hid}/var` — GET, PUT, DELETE `/{name}`

Every mutation records an `InventoryEvent` (`created`, `updated`, `deleted`, `group_added`,
`group_removed`, `host_attached`, `host_detached`, `var_set`, `var_deleted`) via a new
`IInventoryEventService` mirroring `IVmEventService`.

**Before** (today — the only "inventory" is porch's generated single-host line):
```
web-01 ansible_host=10.0.0.12 ansible_user=cloudmanager ansible_ssh_private_key_file=…
```
**After** (C-INV — an addressable inventory record; wire JSON):
```json
{
  "publicId": "inv_AbC123",
  "name": "prod-web",
  "groups": [{ "publicId": "invg_…", "name": "webservers", "parentGroupId": null }],
  "hosts": [{ "publicId": "invh_…", "hostName": "web-01", "virtualMachineId": "vm_…" }]
}
```

> **As-built deviation (P1):** the wire field carrying the public id is `id`, not `publicId`,
> matching the repo-wide CRUD DTO convention (`Blueprint`, `Playbook`, …). Event timeline rows
> keep `publicId` (VmEvent DTO convention). Values are public ids in both cases; raw Guids
> never appear on the wire.

### Feature flag + seed

Migration seeds the `ansible-studio` FeatureFlag row (Enabled=false, Healthy=true — pure
API/data surfaces have no worker dependency yet; P6 adds health probing). Existing `playbooks`
flag untouched.

### MCP (cloud-manager-mcp, contracts §4)

New module `inventory.ts`, registered in profile `hv.cloud`:
`cloud_inventory_list/get/create/update/delete`, `cloud_inventory_group_create/update/delete`,
`cloud_inventory_host_attach/detach`, `cloud_group_var_set/list/delete`,
`cloud_host_var_set/list/delete`. All ids via `idSchema`; camelCase bodies; read/mutate
separation per repo conventions.

### Explicitly out of scope

- No web pages (library UX is P9; deferral recorded in the epic coverage map).
- No change to run targeting: assignments/run targets keep working byte-identically. Wiring
  runs to inventories is P5's operation-context work.
- No var typing/secret handling (P4).

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000 | Setup | hv_status + hv_next; record branch in deck.md |
| 0001 | Entities + migration | 7 entities, prefix registrations, EF configs, migration, `ansible-studio` flag seed |
| 0002 | Services + controllers | DTOs/mappers, CRUD + var upsert + membership, cycle check, flag gating |
| 0003 | Events | `IInventoryEventService`, event writes on every mutation, list endpoint `api/v1/inventory/{id}/event` |
| 0004 | MCP tools | `inventory.ts` module + profile registration + dist rebuild |
| 0005 | Regression + docs | Existing playbook/run flows byte-identical; API smoke; update repo docs |

## Open questions

1. **External (non-VM) hosts now or later?** Default: now, minimally — `invh` allows null
   `virtualMachineId` with explicit `ansibleHost`; nothing executes against them until P5/P6.
2. **Group `all` semantics**: model Ansible's implicit `all` group as a real row per inventory
   or as resolution-time behavior? Default: resolution-time (P5); no magic rows in the DB.
3. **Inventory soft-delete cascade**: block delete while any playbook run references its hosts?
   Default: soft-delete always allowed; history keeps FKs (Restrict on hard paths).
