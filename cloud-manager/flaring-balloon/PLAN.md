# Plan: VM audit log + retry-teardown

## Plan overview

This plan replaces cloud-manager's "force-delete" hammer with two precise tools. A new append-only `vm.vm_events` table becomes the system-of-record for everything that happens to a virtual machine — create, destroy, ack from vorch, IP write-back. A new `retry-teardown` action gives operators a safe way to re-publish `destroy-vm` to vorch when its libvirt cleanup has actually failed, without touching the soft-delete row that PR #19 just made authoritative. When the work is done, the UI shows a per-VM timeline, the force-delete button is gone, and operators have visibility into the destroy lifecycle without reading vorch's `provision.log`.

## Objective

Stand up an append-only VM audit log (`vm.vm_events`) and replace the existing force-delete capability with a `retry-teardown` action that re-publishes `destroy-vm` to vorch. The audit log captures lifecycle events (create, destroy-requested, ack outcomes, IP updates, snapshot ops) and powers a per-VM timeline view. Retry-teardown becomes the only operator action for "vorch's cleanup failed, try again" — and is conditional on the VM's most recent terminal event being `teardown_failed`.

## Background

Earlier in this session we shipped cloud-manager-api PR #19 (`vm-soft-delete`): VMs are now soft-deleted by stamping `deleted_at` instead of being row-deleted. That eliminated the FK-cascade rollback (`playbook_run_targets`, `vm_playbook_assignments` with `ON DELETE RESTRICT`) which had been the main reason force-delete existed — the destroy flow rolled back the DB row even though vorch had already wiped libvirt, leaving the UI showing Running on a VM that no longer existed.

What soft-delete does **not** fix is the rarer failure mode where vorch itself can't reach libvirt and orphans the domain. We observed it in this session on postgres-test (`Could not destroy cloud host: exit status 1`), and `virsh list --all` currently shows three orphan domains (`vm_Z2P33J92PD`, `vm_3BBYD222DE`, `vm_HHYR9RRTB9`) with no matching DB row. Force-delete used to be the only "try again" path; we want a more precise tool.

Separately, the existing destroy flow is hard to debug — there's no per-VM record of when destroy was requested, who requested it, when vorch acked, or what the ack said. The only audit trail is `vorch-service/provision.log`. An events table both fixes the operator-visibility gap and gives us a real data source for retry-teardown's "should this button be visible?" decision.

## Design

### Data model — `vm.vm_events`

```sql
-- AddVmEvents migration
CREATE TABLE vm.vm_events (
    id                  uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
    virtual_machine_id  uuid NOT NULL REFERENCES vm.virtual_machines(id) ON DELETE RESTRICT,
    event_type          varchar(64)  NOT NULL,
    payload             jsonb        NOT NULL DEFAULT '{}'::jsonb,
    actor               varchar(255) NOT NULL,
    occurred_at         timestamptz  NOT NULL,
    public_id           varchar(32)  UNIQUE,
    created             timestamptz  NOT NULL,
    created_by          varchar(255) NOT NULL,
    modified            timestamptz,
    modified_by         varchar(255)
);

CREATE INDEX ix_vm_events_vm_occurred_at  ON vm.vm_events (virtual_machine_id, occurred_at DESC);
CREATE INDEX ix_vm_events_type_occurred_at ON vm.vm_events (event_type,         occurred_at DESC);
```

`event_type` is a string-backed enum at the application layer:

```
created
destroy_requested
teardown_succeeded
teardown_failed
retry_teardown_requested
restored
snapshot_taken
snapshot_reverted
ip_address_updated
playbook_run_started
playbook_run_completed
```

`payload` is jsonb so each event type can carry its own shape (ack body, error message, IP value, snapshot id, playbook run id) without schema churn.

`ON DELETE RESTRICT` is safe because post-soft-delete VM rows are never hard-deleted — the audit row will always have a parent. Records are append-only at the application layer; the service exposes only `RecordEventAsync` and read methods, no Update or Delete.

### EF entity / service

```csharp
// CloudManager.Entities.Models/VmEvent.cs
public class VmEvent : Audit
{
    public Guid     Id                  { get; set; }
    public Guid     VirtualMachineId    { get; set; }
    public string   EventType           { get; set; } = string.Empty;
    public JsonDocument Payload         { get; set; } = null!;
    public string   Actor               { get; set; } = string.Empty;
    public DateTime OccurredAt          { get; set; }
}

// CloudManager.Data.Services/IVmEventService.cs
public interface IVmEventService
{
    Task<VmEvent> RecordEventAsync(Guid vmId, VmEventType type, object payload, string actor, DateTime? occurredAt = null);
    Task<IReadOnlyList<VmEvent>> GetTimelineByVmPublicIdAsync(string publicId, int take = 100);
}
```

DbContext registers `DbSet<VmEvent>`, maps columns, defines both indexes, and registers `public_id` via the existing per-table convention.

### Write path — wired into existing services

| When | Event | Actor | Location |
|------|-------|-------|----------|
| VM create succeeds | `created` | caller | `VirtualMachineService.CreateVirtualMachineAsync` |
| VM destroy requested (soft-delete stamp) | `destroy_requested` | caller | `VirtualMachineService.DeleteVirtualMachineAsync` |
| Retry-teardown invoked | `retry_teardown_requested` | caller | new `VirtualMachineService.RetryTeardownAsync` |
| IP write-back from vorch | `ip_address_updated` | `vorch-service` | `VirtualMachineService.UpdateIpAddressAsync` |
| Vorch destroy ack received | `teardown_succeeded` or `teardown_failed` (mutually exclusive) | `vorch-service` | `CloudManager.AMQP.Consumer` OrchestrationCompletionQueue handler |

`RetryTeardownAsync(string publicId, string actor)` records `retry_teardown_requested`, then publishes `destroy-vm` to RabbitMQ through the exact same producer code the original destroy uses. Vorch is unchanged — it already publishes acks to `OrchestrationCompletionQueue`; this PR consumes them.

### Endpoint contract changes

```
# Before
POST /api/v1/vm/{publicId}/force-delete   # nukes libvirt + DB row, no idempotency
GET  /api/v1/vm/{publicId}                # no event history

# After
POST /api/v1/vm/{publicId}/retry-teardown # republishes destroy-vm to vorch; 400 unless last terminal event is teardown_failed
GET  /api/v1/vm/{publicId}/events?take=100 # per-VM timeline, occurred_at DESC
```

The 400 guard on retry-teardown prevents random retries — operators can only retry when the previous attempt actually failed.

### MCP tool contract change

```
# Before
cloud_vm_force_delete(id: "vm_…")  → wipes libvirt + DB hard delete

# After
cloud_vm_force_delete  REMOVED
```

The retry-teardown action is intentionally not exposed as an MCP tool yet — operator UI only. We can add an MCP tool later if scripted retries become common.

### cloud-manager-web

- New "VM activity" timeline on the VM detail page. Renders the result of `GET /events`: most-recent first, columns `event_type`, `actor`, `occurred_at`, expandable `payload` (jsonb pretty-printed).
- Force-delete button is removed.
- `Retry teardown` button takes its place, visible only when the most recent event on the VM is `teardown_failed`. Disabled otherwise.

## Implementation phases

Each phase produces a green build and (where applicable) a passing test. Phases land in this order:

| Phase | Title | Description |
|-------|-------|-------------|
| 0000  | Setup | hv_status; read PLAN.md + deck.md; build the TaskList |
| 0001  | Schema + entity | `AddVmEvents` migration, `VmEvent` entity, DbContext mapping, public_id wiring |
| 0002  | Service + tests | `IVmEventService` / `VmEventService` with `RecordEventAsync` + `GetTimelineByVmPublicIdAsync` |
| 0003  | Write path — internal | wire `created`, `destroy_requested`, `ip_address_updated` events into existing `VirtualMachineService` methods |
| 0004  | Write path — ack consumer | `CloudManager.AMQP.Consumer` OrchestrationCompletionQueue handler emits `teardown_succeeded` / `teardown_failed` based on ack body |
| 0005  | Retry-teardown service + endpoint | `VirtualMachineService.RetryTeardownAsync`; `POST /api/v1/vm/{publicId}/retry-teardown`; 400 guard on non-`teardown_failed` last event |
| 0006  | Timeline endpoint | `GET /api/v1/vm/{publicId}/events?take=100` |
| 0007  | Remove force-delete from cloud-manager-api | drop endpoint, service method, MCP-facing surface |
| 0008  | Remove force-delete from cloud-manager-mcp | drop `cloud_vm_force_delete` tool registration + handler |
| 0009  | cloud-manager-web — timeline view | per-VM activity panel on VM detail page |
| 0010  | cloud-manager-web — retry-teardown button | replace force-delete button; conditional on last event = `teardown_failed` |
| 0011  | E2E verify | create → destroy → observe `teardown_succeeded`/`teardown_failed`; on failed teardown, click retry, observe new `retry_teardown_requested` + follow-up ack event |
| 0012  | Write Results/RESULT.md + Retro/LESSONS.md; hv_ship |

## Open questions

- Should snapshot and playbook-run event types (`snapshot_taken`, `snapshot_reverted`, `playbook_run_started`, `playbook_run_completed`) be wired in this PR, or deferred? Currently scoped as "wire if the service is on the modify path, otherwise add the enum entry and leave the writer for a follow-up."
- Should the timeline endpoint paginate or just expose `take` for v1? Current proposal: `take` only.
- `actor` resolution — cloud-manager-api currently uses `"admin"` literal for `CreatedBy` (see `VirtualMachineService.CreateVirtualMachineAsync`). Until proper auth lands, events use the same literal for user-initiated actions and `"vorch-service"` for ack-driven events.
