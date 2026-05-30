# Deck: cloud-manager

## Deck name
`cloud-manager`

## Repos with code changes

| Repo | What changes |
|------|-------------|
| `cloud-manager-mcp` | Add 3 new tools to `src/tools/vms.ts` (cloud_vm_update, cloud_vm_ip_address_update, cloud_vm_events). Harden `src/runner.ts` (or shared API-client module) for camelCase-on-wire + 4xx surfacing. Register the new tools in `src/index.ts`. `npm run build` clean. |

Repos not listed will be on the feature branch but skipped by `hv_ship`.

## Branch
`minty-rooster`

## Initialize (Task 0000)
The deck is already on `minty-rooster` from hv_next. Confirm:
```
hv_status  deck: "cloud-manager"
```

## Ship
```
hv_ship  deck: "cloud-manager"
         message: "mcp-coverage-tier-1: cloud_vm_update + ip_address_update + events"
         title:   "mcp coverage tier 1: 3 new VM tools (update, ip-address, events)"
```
