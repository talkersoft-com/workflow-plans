# Plan: cloud-manager-mcp coverage Tier 1 — VM gaps + audit surface

## Plan overview

This plan closes the most-used coverage gaps in cloud-manager-mcp by exposing three VirtualMachine endpoints that existed in cloud-manager-api but had no MCP tool: a generic VM update, the IP-address write-back path that vorch already uses internally, and the audit-log timeline endpoint shipped in the previous PR. When the work is done, a workflow agent can update a VM's display fields, observe its lifecycle timeline, and (if necessary) feed back IP changes — all via typed MCP tools instead of raw curl. The cloud-manager-mcp dist is built and committed but not reloaded in this workflow's session; the operator restarts via /mcp after the PR merges.

## Objective

Add three MCP tools to cloud-manager-mcp matching three existing cloud-manager-api endpoints:

| Tool | Method + Route |
|------|----------------|
| `cloud_vm_update` | `PATCH /api/v1/VirtualMachine/{publicId}` |
| `cloud_vm_ip_address_update` | `PATCH /api/v1/VirtualMachine/{publicId}/ip-address` |
| `cloud_vm_events` | `GET  /api/v1/VirtualMachine/{publicId}/events?take=100` |

Intentionally NOT exposed in Tier 1: `cloud_vm_retry_teardown` (operator-UI only per audit-log PLAN) and `cloud_run_get_secret` (would leak Vault tokens into tool output).

## Background

A coverage analysis (this session) showed cloud-manager-mcp at ~38% endpoint coverage. The biggest immediate gap is the VirtualMachine controller itself: 12 endpoints, only 4 had tools. Tier 1 closes 3 of those 4 (the 4th, retry-teardown, is intentionally out of scope per the audit-log PLAN at workflow-plans/cloud-manager/flaring-balloon/PLAN.md).

The audit-log PR (workflow-plans/cloud-manager/flaring-balloon, merged this session) added two new endpoints (`POST /retry-teardown` and `GET /events`) without corresponding MCP tools. Tier 1 ships the timeline tool now and explicitly defers retry-teardown.

## Design

### Tool: `cloud_vm_update`
- Input: `{ publicId: string, name?: string, description?: string, memory?: number, diskSizeGb?: number, ... }` — only fields permitted by the API's PATCH semantics; do not include `publicId` in the body, do not allow nullable fields the API treats as required.
- Output: the updated VirtualMachine DTO as the API returns it.
- Error mapping: 404 → propagate, 4xx → surface response body in the tool error.

### Tool: `cloud_vm_ip_address_update`
- Input: `{ publicId: string, ipAddress: string }`
- Output: the updated VirtualMachine DTO.
- Note: this endpoint is currently called by vorch-service post-VM-create as well; the MCP tool is for operator overrides and tests.

### Tool: `cloud_vm_events`
- Input: `{ publicId: string, take?: number = 100 }` — server caps at 500.
- Output: array of VmEvent DTOs (occurred_at DESC).

### Cross-cutting: typed wrapper hardening
Before adding the three tools, audit `cloud-manager-mcp/src/runner.ts` (or the shared API client module). If it doesn't already:
1. Refuse to send snake_case keys (camelCase only on the wire per project convention).
2. Surface 4xx `response.error` cleanly.

If those guards are missing, add them. If they're already there, no change needed.

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000  | Setup | hv_status; confirm $NEW_BRANCH across all 15 repos |
| 0001  | Wrapper hardening | audit / harden the shared API client per cross-cutting notes |
| 0002  | cloud_vm_update | add tool, register, build |
| 0003  | cloud_vm_ip_address_update | add tool, register, build |
| 0004  | cloud_vm_events | add tool, register, build |
| 0005  | Coverage assertion | grep test confirming every VM-controller endpoint either has a tool OR is documented as intentionally excluded |
| 0006  | Build + deploy verify per @cloud-manager-deploy | api / web no-op (no changes); mcp dist built; operator /mcp reload reminder in RESULT.md |
| 0007  | Write Results/RESULT.md + Retro/LESSONS.md; hv_ship |

## Open questions

- Should `cloud_vm_update` accept partial payloads or require the full mutable shape? Default: partial; API treats omitted fields as "no change".
- Should `cloud_vm_events` paginate beyond `take`? Default: no, `take` only for v1 (matches the underlying API).
