# Plan: Porch execution upgrades — check mode, ansible-lint, chunked log streaming (ansible-p6-porch)

> Plan 6 of 9 in the Ansible Epic series (windward-steamboat). Conforms to
> `master-plans/ANSIBLE-EPIC-CONTRACTS.md`. **Workflow: `vorch@cloud-manager`** (the only plan
> in the series that changes vorch-service; the vorch workflow rules apply). Satisfies epic §6
> partial (dry-run, lint, streaming output) and §10 (dry-run first, lint gates, idempotency
> visibility). Exposes contract **C-PORCH**. Depends on: none in the series — message-contract
> changes are ADDITIVE ONLY.

## Objective

Give porch three execution-safety capabilities: `--check` dry-runs (same run pipeline, no
mutations on targets), `ansible-lint` runs against materialized playbooks with structured
findings, and chunked log streaming so run output reaches the API as ordered chunks instead of
a truncated 16KB tail. All three ride the existing RabbitMQ + runId machinery; the
`PlaybookRunMessage` and materialization structs in vorch-lib do not change at all.

## Background

- Porch today (`cmd/porch`, `internal/ansible`): consumes `{"runId"}` from `playbook-runs`,
  fetches the run + materialized playbook over HTTP, runs
  `ansible-runner run <workdir> --inventory … -p site.yml --json`, PATCHes per-target status
  with output truncated to the **last 16KB** every 2s, redacts `hvs.*` tokens, two-token Vault
  strategy. Neither `--check` nor `ansible-lint` appears anywhere in the codebase (verified).
- **Vorch workflow rules** (vorch@cloud-manager): message contracts are additive-only — existing
  field names/JSON tags and the five status strings (`queued running succeeded failed
  cancelled`) never change; porch stays a consumer (no request/response over AMQP); crash
  recovery via terminal-status check + nack/redelivery must keep working.
- API-side entities for this plan (`plint`, `logc`) live in cloud-manager-api per contracts §1;
  porch only POSTs to new endpoints.

## Design

### Check mode (epic §10 "dry-run first")

- API: `POST api/v1/playbook/{pid}/run` body gains optional `"mode": "apply" | "check"`
  (additive, default `apply`); persisted as `PlaybookRun.Mode` (additive column, `vm` schema);
  exposed in run DTOs. **No message change** — porch already fetches the run by id and reads
  its fields.
- Porch: when the fetched run has `mode: "check"`, append `--cmdline "--check"` to the
  ansible-runner invocation. Everything else (Vault, inventory, redaction, status writes) is
  identical. Check results land exactly like run results, with `mode` visible so consumers can
  render "would change" semantics (idempotency visibility, epic §10).
- New `vm_events` type (additive): `playbook_check_completed`.

### ansible-lint (epic §7 inline lint source, §10 lint gates)

- API: `POST api/v1/playbook/{pid}/lint` → 202 + runId; new queue **`playbook-lints`** with new
  message `PlaybookLintMessage{ runId }` in vorch-lib (a NEW struct/queue — allowed; existing
  structs untouched). Findings persist to `PlaybookLintResult` (`plint`, per playbook revision,
  jsonb findings in ansible-lint's JSON shape: rule id, severity, path, line, message).
- Porch: new consumer mirrors the collection-install consumer pattern (manual ack, terminal
  check on redelivery). Materializes the playbook (existing helper), runs
  `ansible-lint --format json project/`, POSTs findings to
  `POST api/v1/playbook/{pid}/lint/{runId}/result`, marks the lint run terminal. New
  `vm_events` type (additive, contracts §6): `playbook_lint_completed`.
- Binary health: extend the existing health probe to also probe `ansible-lint --version` and
  PATCH health for the `ansible-studio` flag via the existing
  `…/FeatureFlags/{key}/health` route shape (contracts §11). The `playbooks` probe is untouched.

### Chunked log streaming (epic §6 "stream/chunk, don't buffer")

- API: `PlaybookRunLogChunk` (`logc`, `vm` schema): RunId, TargetId, Seq (monotonic per
  target), Content (≤32KB), CreatedAt. Ingest `POST api/v1/run/{runId}/log/chunk` (worker
  auth); read `GET api/v1/run/{runId}/log/chunk?afterSeq=&max=` for tail-follow consumers
  (P9's run console; MCP `cloud_run_log_chunks` per contracts §4).
- Porch: the existing line-scanner loop additionally flushes redacted output as sequenced chunks
  (per 32KB or 2s, whichever first). **The existing 16KB-tail PATCH behavior is kept** — older
  consumers see identical data; chunks are additive alongside.

**Before** (today — `playbook_run_exec.go`, output truncated to the last 16KB):
```go
// Periodic PATCH payload field: "output" (truncated to last 16KB)
{ "output": "…tail only…", "status": "running" }
```
**After** (C-PORCH — same PATCH still sent, plus ordered chunks, nothing lost):
```json
POST api/v1/run/run_…/log/chunk
{ "targetId": "prt_…", "seq": 17, "content": "TASK [nginx : reload] …\n" }
```

### MCP (contracts §4)

Extends `ansible.ts`: `cloud_playbook_lint` (202+runId), `cloud_playbook_check` (sugar for run
with `mode: "check"`), `cloud_run_log_chunks` (read, paged by afterSeq).

### Additive-only audit (explicit, per vorch rules)

| Surface | Change | Additive? |
|---|---|---|
| `PlaybookRunMessage` / cancel / collection messages | none | — |
| `MaterializedPlaybook` / `MaterializedFile` / `SecretInputRef` | none | — |
| Run status strings | none (lint runs reuse the five) | — |
| New `PlaybookLintMessage` + `playbook-lints` queue | new struct + queue | yes |
| Run DTO | new optional `mode` field | yes |
| Target PATCH | unchanged; chunk POST added beside it | yes |

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000 | Setup | hv_status + hv_next; record branch in deck.md |
| 0001 | API: mode + entities | Run `Mode` column, `plint` + `logc` entities/migrations, lint/chunk endpoints, run DTO additions |
| 0002 | Porch: check mode | `--cmdline "--check"` wiring off fetched run mode; check event; e2e against a real VM |
| 0003 | Porch: lint consumer | vorch-lib `PlaybookLintMessage`, `playbook-lints` queue, lint handler + findings POST |
| 0004 | Porch: chunk streaming | Sequenced chunk flush beside existing tail PATCH; ordering + redaction tests |
| 0005 | Health, MCP, regression | ansible-lint probe → `ansible-studio` health; MCP tools; apply-mode runs byte-identical (tail PATCH parity test) |

## Open questions

1. **ansible-lint installation**: bake into the porch host provisioning (eng-dev-tools/os-image)
   or document as prerequisite with health probe as the guard? Default: health probe guards;
   provisioning addition recorded as a follow-up note, not in this plan's diffs.
2. **Chunk retention**: chunks duplicate the on-disk log file content; prune chunks after run
   terminal + N days? Default: keep 30 days, then prune job (configurable).
3. **Lint gating runs** (epic §10 "lint gates before run"): hard-block runs on lint errors, or
   surface-only? Default: surface-only this plan; gating becomes a P9 guardrail toggle once the
   UI can show findings.
