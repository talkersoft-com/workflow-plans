# Findings — vorch/porch status surface vs. the Marketplace UI (copper-kiosk Phase 0005)

## Verdict

**Sufficient.** The Marketplace UI (storefront, builder, VM provenance + live run-chain panel)
was built and verified end-to-end against the live backend with **zero changes** to
cloud-manager-api, cloud-manager-mcp, vorch-lib, vorch-service, or porch. No follow-up
`vorch@cloud-manager` plan is required, so no draft is included. One client-side compromise and
two non-blocking observations are documented below.

## Question A — Provision visibility ("watching my VM come up")

**Verdict: sufficient.**

- Orchestration status writeback: the create-vm ack flips `OrchestrationStatus`
  Initialized → Completed/Failed in one step (`ConsumerService.cs` case `create-vm`) and
  broadcasts `OrchestrationStatusUpdated` to all SignalR clients. The existing
  `signalRMiddleware` refetches VM state on that broadcast, so the VM detail page flips
  Provisioning → Running live. There is no intermediate "InProgress" broadcast, but the UI never
  needed one: the page already communicates the in-flight phase with its Provisioning status.
- IP writeback: `ipAddress` is null at create (observed on `vm_N55GHGVC9Z` immediately after
  instantiate) and appears with the vorch IP writeback before the ack completes the status flip.
  Observed end-to-end: instantiated `jammy` VM showed Running + IP `10.0.150.66` ~90 s after the
  three-click create, no manual refresh.

## Question B — Chain visibility (run-chain panel)

**Verdict: composable from existing endpoints — with one client-side fix (FIX-001) and one
documented compromise.**

- **What did NOT work as planned:** deriving step state from
  `PlaybookAssignment.lastRunStatus` (`GET /virtualmachine/{id}/playbook`). The API writes that
  field only on the run-target writeback (`PlaybookRunTargetService.cs:115`), not when the
  sequencer enqueues a run — so the entire Queued phase is invisible through assignments
  (observed live: `run_V9AV4BMRNX` Queued while the assignment still said `None`).
- **What works:** `GET /api/v1/run?vmId={publicId}` (the AnsibleHistoryPage endpoint) exposes
  the newest run per playbook with full status (Queued/Running/Succeeded/Failed/Cancelled) and
  the run id for the failure deep-link. The panel composes: `blueprint_instantiated` VmEvent
  (provenance + runPlan snapshot) + assignments (runPlan → playbook ids) + blueprint detail
  (order, names) + run list (live status). See `Retro/FIX-001.md`.
- **SignalR coverage — the gap and the compromise:** the hub only emits
  `OrchestrationStatusUpdated` on create-vm acks and `LogLine` to per-run log groups. There is
  **no broadcast** when the sequencer enqueues a run or when a target reaches a terminal state.
  A purely event-driven chain panel is therefore impossible on the current surface. The panel
  uses exactly the RunDetailPage pattern: hub-triggered refetch where broadcasts exist, plus a
  **bounded** 2.5 s refresh that runs only while the chain is non-terminal and stops at
  Succeeded/Failed. RunDetailPage itself live-updates run status the same way (its
  "Phase 12 — live polling while run is Queued or Running" block), so this matches existing app
  convention rather than introducing a new one.
- **SSH-wait window honesty:** between provision-complete and first enqueue (~38 s observed),
  the step shows **Pending** — an honest "not started yet". Evidence (recorded in-browser, no
  reload): `postgres-jammy` chain on `vm_JYTZEWWBWM`: Pending (t=0) → Queued (t=38 s) →
  Succeeded (t=125 s). Failure path on `vm_T9D5Z4D8SZ` (always-fail fixture): Pending →
  Failed + "Chain stopped here" + link to `/ansible/runs/run_3R5DPA0NS0` at t=54 s.

## Non-blocking observations (no plan drafted — nothing here is required by the shipped UI)

1. A hub broadcast on run enqueue / target terminal writeback (additive, e.g. reusing
   `OrchestrationStatusUpdated` with a `playbook-run` command payload) would let the chain panel
   drop its bounded poll entirely. Pure improvement; current UX meets the spec without it.
2. A `blueprint_chain_waiting_for_ssh` VmEvent (or similar) would let the panel distinguish
   "VM still provisioning" from "SSH wait" during the Pending phase. Cosmetic.

## Evidence index

- Live transition log (TC-001, Phase 4): `Retro/FIX-001.md` + Test 0004 records in
  `Results/RESULT.md` (Phase 6).
- Hub surface: `ConsumerService.cs:142` (single broadcast site), `RunLogConsumerService.cs:55`
  (log groups), `NotificationHub.cs`.
- Writeback site: `PlaybookRunTargetService.cs:115`.
- Sequencer state derivation: `BlueprintRunSequencer.cs:203-213` (derives from
  `LastRunStatus`, consistent with the above).
