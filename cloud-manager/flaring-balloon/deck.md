# Deck: cloud-manager

## Deck name
`cloud-manager`

## Repos with code changes

| Repo | What changes |
|------|-------------|
| `cloud-manager-api` | `AddVmEvents` migration; `VmEvent` entity + DbContext mapping; `VmEventService` + interface; wire `created` / `destroy_requested` / `ip_address_updated` events into `VirtualMachineService`; AMQP consumer emits `teardown_succeeded` / `teardown_failed` from OrchestrationCompletionQueue ack; `RetryTeardownAsync` + `POST /api/v1/vm/{publicId}/retry-teardown` endpoint (400 unless last terminal event is `teardown_failed`); `GET /api/v1/vm/{publicId}/events?take=100` endpoint; remove force-delete endpoint + service method. |
| `cloud-manager-mcp` | Drop `cloud_vm_force_delete` tool registration + handler. |
| `cloud-manager-web` | Remove force-delete button; add `Retry teardown` button conditional on last event = `teardown_failed`; new "VM activity" timeline view on VM detail page driven by `GET /events`. |

Repos not listed will be on the feature branch but skipped by `hv_ship`.

## Branch
`flaring-balloon`

## Initialize (Task 0000)
The deck is already on `flaring-balloon` from the planning ship. Confirm:
```
hv_status  deck: "cloud-manager"
```
If somehow not on `flaring-balloon`:
```
hv_next  deck: "cloud-manager"
```

## Ship
```
hv_ship  deck: "cloud-manager"
         message: "vm-events: append-only audit log + retry-teardown action; remove force-delete"
         title:   "vm-events: per-VM audit log + retry-teardown, retire force-delete"
```
