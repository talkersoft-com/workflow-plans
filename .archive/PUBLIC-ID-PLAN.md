# Public-ID Plan: Stripe/AWS-Style Identifiers for API Entities

## Problem

Every entity today exposes its raw `Guid Id` to API consumers, and several entities (notably `VirtualMachine`) reuse the human-friendly `Name` field as the identifier in external systems:

- libvirt domain name = `VirtualMachine.Name`
- libvirt domain xml file path = `/etc/libvirt/qemu/<Name>.xml`
- Vault SSH key path = `cloudmanager/vm/instances/<Name>`
- AMQP payload key = `vmname` (same `Name`)
- Disk image filename = derived from `Name`

This couples three orthogonal concerns into one mutable, user-controlled string:

1. **Display** — what humans see in the UI.
2. **System identity** — what libvirt, Vault, and AMQP key off.
3. **API identity** — what URLs are built from.

The consequences at scale:

- **Renames are impossible.** Changing `Name` would orphan Vault secrets, break libvirt lookups, and corrupt every downstream join.
- **Collisions are user-controlled.** Two organizations creating "test" or "db" conflict at the libvirt/Vault layer.
- **Raw user input flows into shell commands and filesystem paths.** Anywhere `Name` is interpolated into a `virsh` invocation or Vault HTTP call is a latent shell-/path-injection surface.
- **UUIDs leak.** Internal `Guid Id`s appear in API URLs (`/api/v1/virtualmachine/get/{guid}`), tying URL stability to PK choice and giving consumers an unstable mental model.

## Goal

Introduce a Stripe/AWS-style **public identifier** per entity (`vm_8x3k2pm9wq`, `snap_…`, `host_…`, etc.). Use this identifier consistently as the **API contract** *and* as the system-level identity in libvirt and Vault. Free the `Name` field to be pure mutable display metadata.

---

## Identifier Model

| Layer | Identifier | Mutable | Notes |
|---|---|---|---|
| DB primary key / FK joins | `Guid Id` | no | Internal only; never on the wire. |
| libvirt `<uuid>` element | `Guid Id` | no | Libvirt's own UUID slot, matches DB PK. |
| libvirt `<name>` element | `PublicId` | no | What `virsh list` shows. |
| libvirt `<title>` / `<description>` | `Name` | yes | Display string in `virsh dominfo`. |
| Disk image filename | `PublicId` | no | `/var/lib/libvirt/images/<public_id>.qcow2`. |
| Vault secret path | `cloudmanager/vm/instances/<PublicId>` | no | Stable secret location. |
| AMQP payload identifier | `PublicId` | no | Replaces `vmname` field. |
| API URLs and DTOs | `PublicId` (serialized as `id`) | no | Guid is never serialized out. |
| Display | `Name` | **yes** | Pure metadata, can be edited freely. |

### PublicId format

- Pattern: `<prefix>_<10 chars Crockford base32>` — e.g. `vm_8x3k2pm9wq`.
- Crockford base32 alphabet (`0123456789ABCDEFGHJKMNPQRSTVWXYZ`) — no `I`, `L`, `O`, `U`, so typeable and unambiguous when read aloud.
- 50 random bits → ~1.1 quadrillion possible values → effectively no collision risk at expected scale, but a unique index will still catch the impossible.
- Prefix per CLR type (case-sensitive, lowercase):

  | Entity | Prefix |
  |---|---|
  | `VirtualMachine` | `vm` |
  | `VirtualMachineSnapshot` | `snap` |
  | `VirtualMachineImage` | `vmi` |
  | `Host` | `host` |
  | `DesignatedHost` | `dhost` |
  | `Organization` | `org` |
  | `User` | `usr` |
  | `Team` | `team` |
  | `Role` | `role` |
  | `IsoFile` | `iso` |
  | `NetworkAddress` | `naddr` |
  | `NetworkConfiguration` | `ncfg` |
  | `OrchestrationCommand` | `ocmd` |
  | `VirtualDevice` | `vdev` |

---

## Phase 1: Schema & Generation

### 1.1 Add `PublicId` to the `Audit` base class

```csharp
public abstract class Audit
{
    public string PublicId { get; set; } = string.Empty;
    // existing audit fields (Created, CreatedBy, Modified, ModifiedBy)
}
```

Every entity already inherits from `Audit`, so they all gain the column for free.

### 1.2 EF Core configuration (per entity in `CloudManagerDbContext`)

```csharp
entity.Property(e => e.PublicId)
      .HasMaxLength(32)
      .IsRequired();
entity.HasIndex(e => e.PublicId).IsUnique();
```

### 1.3 Generation interceptor

Add a `SaveChangesInterceptor` that, on insert, fills `PublicId` if empty:

- Resolve prefix from a `Dictionary<Type, string>` keyed on CLR type (see table above).
- Sample 50 random bits via `RandomNumberGenerator.GetBytes(7)`.
- Encode with Crockford base32 → 10 chars.
- Concatenate `<prefix>_<suffix>`.

A `ValueGenerator` is the EF Core idiom for this, but an interceptor keeps the prefix-per-type logic in one place rather than scattered across entity configs.

### 1.4 Migration

- Add nullable `public_id varchar(32)` column to every `Audit` table.
- Backfill existing rows (see Phase 6 below).
- Once backfilled, add `NOT NULL` + unique index in a follow-up migration.

---

## Phase 2: API Contract Changes

### 2.1 DTOs

`PublicId` becomes the wire format for `id`:

```csharp
// CloudManager.DTO.Models.VirtualMachine
[JsonPropertyName("id")]
public string Id { get; set; } = string.Empty;       // was Guid
// internal Guid no longer serialized
```

Mapping layer (`CloudManager.Data.Services`) projects entity → DTO with `dto.Id = entity.PublicId`.

### 2.2 Route changes

`VirtualMachineController` (and all sibling controllers) accept the public id, not the Guid:

```csharp
[HttpGet("get/{publicId}")]
public Task<IActionResult> GetAsync(string publicId) { ... }

[HttpPost("create/{hostPublicId}")]
public Task<IActionResult> CreateAsync(string hostPublicId, ...) { ... }
```

Service-layer methods grow `…ByPublicIdAsync` variants. Internal joins continue to use Guid; controllers translate at the edge.

### 2.3 Cross-entity references in request bodies

Where a request body currently sends a Guid (e.g. `VirtualMachineImageId`, `HostId`, `ActiveSnapshotId`), switch to the public id of the referenced entity. The service layer resolves to Guid via the unique index.

### 2.4 Backward compatibility

None. This is a breaking API change. cloud-manager-web is the only consumer and will be updated in lockstep. Document the cut in `README.md`.

---

## Phase 3: AMQP Payload Changes

### 3.1 Replace `vmname` with `public_id`

`MessagePayload<CreateVmCommandData>.Data.Vm`:

```yaml
vm:
  public_id: vm_8x3k2pm9wq        # was: vmname
  display_name: my-prod-db        # new: free-form metadata, informational only
  memory: 2048
  ...
```

- `public_id` is the system identifier vorch-service uses for libvirt domain, Vault path, disk filename.
- `display_name` exists only so vorch-service can stamp it into libvirt `<title>` for `virsh dominfo` readability. Never used as a key.

Same shape for `create-snapshot`, `destroy-vm`, snapshot revert, and any future command — all key off `public_id`.

### 3.2 OrchestrationCompletionQueue replies

Already keyed by `Identifier = virtualMachine.Id` (Guid). Keep as Guid — the consumer (`CloudManager.AMQP.Consumer.ConsumerService`) is internal and joins to the DB on PK. No change needed here.

---

## Phase 4: vorch-service / vorch-lib Changes

### 4.1 Domain creation

In `vorch-lib/provision/provision.go`:

- libvirt XML `<name>` = `public_id`
- libvirt XML `<uuid>` = the Guid embedded in the AMQP payload (new field — pass through from API)
- libvirt XML `<title>` = `display_name`

### 4.2 Disk image path

`/var/lib/libvirt/images/<public_id>.qcow2` instead of `<vmname>.qcow2`.

### 4.3 Vault secret path

`cloudmanager/vm/instances/<public_id>` instead of `cloudmanager/vm/instances/<vmname>`. Already the path shape — only the value of the trailing segment changes.

### 4.4 Destroy / snapshot handlers

All `virsh` invocations switch from `<vmname>` to `<public_id>`. Same for `vault kv delete` on VM destruction.

### 4.5 Input validation

`public_id` must match `^[a-z]+_[0-9A-HJKMNPQRSTVWXYZ]{10}$`. Reject anything else before it reaches shell commands — closes the injection surface we currently have via free-form `Name`.

---

## Phase 5: Frontend Changes (cloud-manager-web)

- All list/detail/edit views key off the new `id` (which is `public_id`).
- URLs become `/vm/vm_8x3k2pm9wq` instead of `/vm/<guid>`.
- VM rename UI now edits `name` (display) freely; `id` is read-only.

---

## Phase 6: Data Migration

### 6.1 Backfill existing rows

For each `Audit`-derived table with existing rows:

1. Compute `public_id` for each row using the same generator (resolve prefix by table).
2. `UPDATE … SET public_id = $1 WHERE id = $2` per row.
3. Verify count: every row has a non-null, well-formed `public_id`.
4. Run the follow-up migration to add `NOT NULL` + unique index.

### 6.2 Existing libvirt domains

For each VM currently running on the host:

1. Build a manifest: `{ db_id, db_name, new_public_id }`.
2. For each: `virsh dumpxml <db_name>` → edit `<name>` → `virsh undefine <db_name>` → `virsh define <new.xml>`.
3. Rename disk image: `mv /var/lib/libvirt/images/<db_name>.qcow2 /var/lib/libvirt/images/<public_id>.qcow2`.
4. Update VM XML to point at the renamed disk.

This is a one-shot migration script delivered under `cloud-manager-api/scripts/migrations/public-id-backfill/`. Must run with the orchestration service stopped to avoid races.

### 6.3 Existing Vault secrets

For each VM with a key currently at `cloudmanager/vm/instances/<db_name>`:

1. `vault kv get -format=json cloudmanager/vm/instances/<db_name>` → capture value.
2. `vault kv put cloudmanager/vm/instances/<public_id>` ← same value.
3. `vault kv delete cloudmanager/vm/instances/<db_name>`.

Verify by reading the new path before deleting the old.

### 6.4 Rollback plan

- The migration script writes a manifest to disk (`migration-state.json`) before any mutation.
- A `--rollback` flag reads the manifest and reverses each step (libvirt rename back, vault path rename back). DB rollback handled by EF Core migration `Down()`.

---

## Phase 7: Rollout Order

The phases above are not safe in arbitrary order. Correct ordering:

1. **Schema migration** (Phase 1.1–1.2) — add nullable column, no behavior change.
2. **Generator** (Phase 1.3) — new entities get a `public_id`. Old ones remain null.
3. **Backfill script run** (Phase 6.1) — every row now has a `public_id`.
4. **NOT NULL + unique migration** (Phase 1.4) — enforces invariant going forward.
5. **vorch-service update** (Phase 4) — deployed with **dual-read** logic: accepts AMQP payloads with either `vmname` or `public_id` for a transition window. Writes use `public_id` for everything new.
6. **Existing libvirt + Vault data migration** (Phase 6.2–6.3) — one-shot script, orchestration paused.
7. **cloud-manager-api update** (Phase 2 + 3) — controllers and AMQP publishers switch to `public_id`. `vmname` field dropped from payload schema.
8. **cloud-manager-web update** (Phase 5) — UI consumes new contract.
9. **vorch-service follow-up** — remove dual-read fallback; `public_id` becomes required.

Each step is independently deployable. Steps 5–9 must happen in roughly this order, but with the dual-read window in step 5 there is no hard cutover moment.

---

## Resolved Decisions

- **`Host`, `Organization`, and every other `Audit`-derived entity get the same treatment.** The generator and migration walk every table; there is no critical-path-only carve-out. Drives the prefix table in the Identifier Model.
- **Audit logs key on `public_id` only.** When the audit table format is revisited, drop the Guid column from the audit-log surface. The `public_id` is stable, immutable, and cross-references everywhere; carrying the Guid alongside is duplicate noise. The Guid still exists on the source row for FK joins — it just doesn't propagate into audit records.

## Out of Scope (Tracked Elsewhere)

- **vorch-service Vault auth method (AppRole / token renewal).** Explicitly **not addressed by this plan**. The current static token will continue to be used; expiry mitigation is left to a future, separate effort. (Earlier drafts of this plan cited `VAULT-SSH-KEY-PLAN.md` as owning this; that doc does not actually plan an auth-method change — reference removed.)
- **Multi-tenant scoping of `public_id` uniqueness.** Public IDs are globally unique under this plan. When tenants are introduced, the unique index becomes `(tenant_id, public_id)` and the generator can drop a bit or two of entropy in favor of namespacing. Deferred until the tenant model lands; not blocking here.
