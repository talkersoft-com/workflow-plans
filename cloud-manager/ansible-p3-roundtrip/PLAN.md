# Plan: Round-trip YAML service — decompose/recompose with full fidelity (ansible-p3-roundtrip)

> Plan 3 of 9 in the Ansible Epic series (windward-steamboat). Conforms to
> `master-plans/ANSIBLE-EPIC-CONTRACTS.md`. Workflow: `feature@cloud-manager`.
> Satisfies epic §2 (round-trip fidelity: comments, key order, formatting preserved).
> Exposes contract **C-RT**. Depends on: P2 (round-trips the decomposed record model).

## Objective

Add a round-trip-preserving YAML service built on `ruamel.yaml` so that decompose → store →
recompose returns byte-identical YAML — comments, key order, anchors, block-scalar styles and
all. P2's YamlDotNet decomposer is deliberately lossy (read model only); this plan adds the
*write path*: structured edits to plays/tasks/vars that splice back into the user's YAML without
mangling formatting. This is the invariant (epic §11 "data integrity") every later editing
feature (P7 editor save, P9 composer) relies on.

## Background

- `ruamel.yaml` is Python; the API is C# and porch is Go — the round-trip engine cannot be
  in-process with either. Where it runs is this plan's REQUIRED decision (below).
- P2 stance recap: YAML text is the source of truth; records are derived. P3 does not change
  that — it makes targeted text *edits* through a structure-aware engine, then lets P2's
  decomposer re-derive records from the new text.

### REQUIRED decision: sidecar vs porch helper — **decision: API sidecar**

| | API sidecar (chosen) | porch helper (rejected) |
|---|---|---|
| Latency | synchronous HTTP, same host as API — fits interactive editing | RabbitMQ round-trip per edit — seconds, not ms |
| Coupling | authoring concern stays in the authoring tier | drags porch into request/response work it was built to avoid |
| Security | no Vault access needed; sidecar sees YAML text only | porch holds worker Vault tokens — expanding its surface is the wrong direction |
| Message contract | none touched | would demand new queues/messages (P6's additive-only budget spent on editing, not execution) |
| Ops | one new container next to the API | porch deploys on hypervisor hosts — editing would depend on hypervisor availability |

Rationale: round-trip is a synchronous, per-keystroke-adjacent authoring operation; porch is an
async execution worker on a different host with a deliberately minimal, additive-only contract.
A small Python sidecar (FastAPI + ruamel) deployed beside the API keeps editing latency
interactive and leaves porch untouched.

## Design

### Round-trip sidecar

New `roundtrip-service/` folder in the cloud-manager-api repo (Python 3.12, FastAPI, ruamel;
own Dockerfile, deployed as a container next to the API; not network-exposed beyond the API).
Endpoints (internal):

- `POST /decompose` — YAML text → structural map with per-node source spans + style metadata
- `POST /recompose` — original text + list of structured edits (set/insert/delete at record
  paths) → new YAML text, formatting preserved
- `GET /healthz`

The API proxies these as `api/v1/roundtrip/decompose|recompose` (contracts §3), auth-scoped,
flag-gated `ansible-studio`; web/MCP never call the sidecar directly.

### YamlDocument (`ydoc`, schema `ansible`, contracts §1)

Per `PlaybookRevision`/`RoleFileRevision`: canonical source text checksum + the sidecar's
decomposition map (jsonb). Written whenever a revision is cut. Purpose: (a) proof of fidelity —
`recompose(decompose(text)) == text` checked at write time, mismatch fails loudly; (b) the
span map that P2 records and P8 decorations anchor to without re-parsing.

### Structured-edit write path

`PATCH api/v1/playbook/{pid}/play/{playId}` and `…/task/{ptaskId}` (the endpoints P2 deferred):
accept structured field changes → API builds a recompose edit list → sidecar splices → new text
is saved through the *existing* revision-cutting path (so `pbrev`/`rfrev` history, P2
re-decomposition, and events all fire exactly as a raw-text save would).

**Before** (today — any programmatic edit re-serializes and destroys formatting):
```yaml
# deploy web tier   ← comment lost
- name: deploy
  hosts: webservers   # key order + this comment lost on re-serialize
```
**After** (C-RT — set `hosts: all` via structured edit; everything else byte-identical):
```yaml
# deploy web tier
- name: deploy
  hosts: all   # key order + this comment lost on re-serialize
```

### Fidelity test corpus

Golden-file suite: real-world playbooks (anchors/aliases, block scalars `|`/`>`, deep comments,
flow style, weird indentation) → assert `recompose(decompose(x)) == x` byte-identical, and each
structured edit changes only the intended span. Corpus lives in the sidecar's tests; CI-runnable
with plain pytest.

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000 | Setup | hv_status + hv_next; record branch in deck.md |
| 0001 | Sidecar service | FastAPI + ruamel decompose/recompose/healthz, Dockerfile, golden-file corpus |
| 0002 | ydoc + proxy | `ydoc` entity + migration; API proxy routes; write-time fidelity check on revision cut |
| 0003 | Structured edits | PATCH play/task endpoints → edit list → splice → existing revision path |
| 0004 | P2 integration | Decomposer prefers sidecar span map when present; spans flow into `ptask.SourceSpan` |
| 0005 | Regression + docs | Raw-text save path byte-identical; corpus green; deployment docs |

## Open questions

1. **Sidecar deployment unit**: docker-compose alongside the API container vs systemd service?
   Default: same compose stack as the API (one `docker compose up` story).
2. **Recompose conflict handling**: if the revision changed since the edit list was built (span
   drift), reject with 409 or rebase? Default: 409 + client refetch — simplest correct behavior.
3. **Sidecar language tooling**: uv + ruff to match modern Python practice? Default: yes, but
   zero impact outside `roundtrip-service/`.
