# Playbook Integration Plan

## Problem

Beyond cloud-init (one-shot at first boot), Cloud Manager has no way to
install or configure software on a VM. We want users to be able to deploy
postgres, rabbit, nginx, haproxy, etc. without writing a new code path in
Cloud Manager for each appliance — and without owning the recipe code
ourselves.

Two real risks of the "build it ourselves" approach:

- Every new appliance becomes maintenance work in our codebase.
- The recipes need versioning, parameter validation, idempotency, retries,
  rollback — all of which Ansible already does.

## Goal

Generic Ansible playbook execution: assign **any** playbook to a VM with
typed parameters, apply it on demand, surface generated secrets back to the
user — **with zero playbook-specific code in Cloud Manager**. Everything
playbook-specific lives in the playbook repo, behind two conventions:
`meta/argument_specs.yml` (Ansible-native, declares inputs) and
`meta/outputs.yml` (our convention, declares what secrets/info the
playbook produces).

## Core Principles

1. **No playbook-specific code in this repo.** Cloud Manager treats every
   playbook the same; differences are expressed in playbook-side YAML.
2. **Inputs are typed.** Each playbook declares its inputs in
   `meta/argument_specs.yml`. We cache it as JSON Schema and render a form.
3. **Outputs are conventional.** Each playbook writes its secrets under a
   single Vault prefix we hand it (`secrets_prefix` extra-var). We list
   that prefix after a run and surface the contents.
4. **Execution via `ansible-runner`** — the Python lib that powers AWX
   internally. Not AWX itself (that would replace this app).
5. **Per-run scoped Vault tokens.** Orchestrator mints a short-lived child
   token scoped to the (vm, playbook) Vault prefix. Token dies with the
   run.
6. **Opt-in.** The feature is off by default. An installation without
   Ansible should look like the feature doesn't exist — no broken
   endpoints, no greyed-out menu items. A DB-backed flag turns it on once
   `ANSIBLE-INSTALL-PLAN.md` has been completed, and a periodic health
   probe distinguishes "enabled but broken" from "enabled and working."

## Feature Flag

A generic `feature_flags` table (not playbook-specific — there will be
more flags later: per-org Vault overrides, alternate auth providers,
etc.):

```
feature_flags (membership schema — closest existing home)
─────────────
id            uuid PK
public_id     varchar(32)  ff_<10 Crockford>
key           varchar(64)  unique  e.g. "playbooks"
enabled       bool                 default false
healthy       bool                 default false  set by health probe
last_probed_at timestamptz         when health was last checked
description   text                 human-readable purpose
+ Audit
```

`EntityPrefixRegistry` addition: `ff`.

### Three states, three behaviors

| State                       | API                                 | Web UI                                                  |
|-----------------------------|-------------------------------------|---------------------------------------------------------|
| `enabled=false`             | All `/playbook/*` routes return 404 | Playbooks section completely hidden                     |
| `enabled=true, healthy=true`| Routes work normally                | Playbooks section visible and functional                |
| `enabled=true, healthy=false`| Routes return 503 with diagnosis    | Playbooks section visible but banner: "ansible-runner not reachable — see install plan" |

404 (not 503) when disabled is deliberate: it should look like the feature
doesn't exist to outside callers, matching the "opt-in feature" promise.

### `/feature-flags` endpoint

```
GET /api/v1/feature-flags
→ { "playbooks": { "enabled": true, "healthy": true, "last_probed_at": "..." } }
```

Web UI fetches this on app load and gates entire sections off it. No
per-feature feature-flag knowledge in the React components — they read a
single map.

### Admin toggle

```
PATCH /api/v1/admin/feature-flags/{key}    body: { enabled: bool }
```

Authenticated. (Auth model deferred — out of scope for this plan; for now
behind whatever auth Cloud Manager grows into.)

## Data Model Additions

In addition to `feature_flags` above, three new tables in the `vm`
schema, all inheriting `Audit`:

### `playbooks`

```
id                   uuid PK
public_id            varchar(32)  pb_<10 Crockford>
name                 varchar(255) unique
description          text
git_url              varchar(2048)
git_ref              varchar(255)  branch / tag / SHA
playbook_path        varchar(255)  e.g. "site.yml", "playbooks/install.yml"
argument_schema      jsonb         cached from meta/argument_specs.yml at git_ref
output_schema        jsonb         cached from meta/outputs.yml at git_ref (optional)
last_refreshed_at    timestamptz
+ Audit (public_id, created, created_by, modified, modified_by)
```

### `vm_playbook_assignments`

```
id                   uuid PK
public_id            varchar(32)  asgn_<10 Crockford>
virtual_machine_id   uuid FK -> vm.virtual_machines
playbook_id          uuid FK -> vm.playbooks
vars_override        jsonb         user-edited params, validated against playbook.argument_schema
last_run_id          uuid FK -> vm.playbook_runs (nullable)
last_run_status      enum (None, Running, Succeeded, Failed)
last_applied_at      timestamptz
+ Audit
UNIQUE (virtual_machine_id, playbook_id)
```

### `playbook_runs`

```
id                    uuid PK
public_id             varchar(32)  run_<10 Crockford>
assignment_id         uuid FK -> vm.vm_playbook_assignments
status                enum (Queued, Running, Succeeded, Failed, Cancelled)
started_at            timestamptz
completed_at          timestamptz
stats                 jsonb  ansible-runner host stats (ok, changed, failed, etc.)
vault_secrets_prefix  varchar(512)  the path the orchestrator scanned for outputs
secrets_published     jsonb  [{ path, output_key, type, description }]
error_message         text
+ Audit
```

`EntityPrefixRegistry` additions: `pb`, `asgn`, `run`.

`stdout_log` deliberately not modeled as a column — see "Run Log Storage"
below.

## Schema Conventions

### Inputs: `meta/argument_specs.yml`

This is Ansible-native (since 2.11), readable by `ansible-doc`, and used
by any decent role. We cache it as JSON Schema in
`playbooks.argument_schema`.

```yaml
argument_specs:
  main:
    short_description: Install and configure PostgreSQL
    options:
      postgres_version:
        type: int
        default: 15
        choices: [13, 14, 15, 16]
        description: Major version to install
      postgres_listen_addresses:
        type: str
        default: "localhost"
      postgres_databases:
        type: list
        elements: dict
        options:
          name:  { type: str, required: true }
          owner: { type: str, required: true }
```

### Outputs: `meta/outputs.yml` *(Cloud Manager convention, not Ansible)*

Tells us what to expect under the Vault prefix after a run. We list/scan
that prefix anyway, but the schema gives us labels, descriptions, and
type hints for the UI.

```yaml
outputs:
  postgres_admin_password:
    type: secret
    relative_path: postgres/admin_password
    description: PostgreSQL admin (postgres user) password
  postgres_connection_uri:
    type: secret
    relative_path: postgres/connection_uri
    description: psql connection URI
  postgres_port:
    type: info
    relative_path: postgres/port
    description: Service port
```

If `meta/outputs.yml` is absent, we still list the prefix after a run and
display every key found, just without nice labels.

## Predictable Vault Path Pattern

```
cloudmanager/
└── vm/
    └── instances/
        └── {vm_public_id}/
            ├── ssh_key                          ← existing (Phase 4)
            └── {playbook_public_id}/
                └── {output relative_path}       ← playbook-written secrets
```

Concrete example for a postgres playbook run on `vm_8x3k2pm9wq` using
playbook `pb_K2QMNPWVR0`:

```
cloudmanager/vm/instances/vm_8x3k2pm9wq/pb_K2QMNPWVR0/postgres/admin_password
cloudmanager/vm/instances/vm_8x3k2pm9wq/pb_K2QMNPWVR0/postgres/connection_uri
cloudmanager/vm/instances/vm_8x3k2pm9wq/pb_K2QMNPWVR0/postgres/port
```

This is fully predictable from public_ids and the convention — no DB
lookup needed to reconstruct where a secret lives.

## API Surface

Every `/api/v1/playbook/*` and every per-VM playbook route checks
`feature_flags.playbooks.enabled` first. Returns 404 when disabled, 503
with diagnosis when enabled-but-unhealthy. Controllers use a simple
`[RequireFeatureFlag("playbooks")]` attribute.

```
# Feature flags (always available)
GET    /api/v1/feature-flags                       all flags, web reads at app load
PATCH  /api/v1/admin/feature-flags/{key}           toggle enabled (admin)

# Playbook registry (404 unless flag enabled)
GET    /api/v1/playbook                            list registered playbooks
POST   /api/v1/playbook                            register {name, git_url, git_ref, playbook_path}
GET    /api/v1/playbook/{publicId}                 get one (incl. argument_schema, output_schema)
PATCH  /api/v1/playbook/{publicId}                 update git_ref / name; triggers refresh
POST   /api/v1/playbook/{publicId}/refresh         re-fetch schemas from git_ref
DELETE /api/v1/playbook/{publicId}                 (rejects if any assignments exist)

# Assignments (VM ←→ playbook)
GET    /api/v1/virtualmachine/{vmId}/playbook
POST   /api/v1/virtualmachine/{vmId}/playbook                attach {playbookId, vars_override}
PATCH  /api/v1/virtualmachine/{vmId}/playbook/{assignId}     update vars_override
DELETE /api/v1/virtualmachine/{vmId}/playbook/{assignId}

# Execution
POST   /api/v1/virtualmachine/{vmId}/playbook/{assignId}/apply   → returns run_id immediately
GET    /api/v1/run/{runId}                                       status + stats + secrets_published
GET    /api/v1/run/{runId}/log                                   full stdout (paginated)
# SignalR hub /notificationHub: channel "RunLog_<run_id>" streams stdout lines live
```

## Execution Flow

When the user clicks "Apply" on an assignment:

1. **API**: insert `playbook_runs` row (status=Queued), publish a message
   to RabbitMQ queue `playbook-runs` with `{run_id, assignment_id}`.
   Respond 202 with run_id.
2. **Worker** consumes message:
   1. Load assignment, playbook, VM, host, network address from DB.
   2. Validate `vars_override` against current `argument_schema`; refresh
      from git if stale and re-validate. Fail Run if schema diverged.
   3. Build `secrets_prefix` = `cloudmanager/vm/instances/{vm_pub}/{pb_pub}`.
   4. Mint a Vault child token (TTL 1h, policy
      `cloudmanager-playbook-runner` — see install plan). Token allows
      `create/update/read/list` under `secrets_prefix` only.
   5. Pull per-VM SSH private key from Vault, write to tmpfile (0600).
   6. Resolve any input vars flagged as secrets in `argument_schema`
      (`x-secret-path: cloudmanager/...`) using orchestrator's own token,
      merge resolved values into extra-vars.
   7. `git clone --depth 1 --branch {git_ref} {git_url}` into ephemeral
      tmpdir on tmpfs.
   8. Generate `ansible-runner` private_data_dir:
      - `inventory/hosts.yml` with this one VM
      - `env/extravars` = `{ secrets_prefix, vm_public_id, playbook_public_id, ...vars_override, ...resolved_secret_inputs }`
      - `env/envvars` = `{ VAULT_ADDR, VAULT_TOKEN: <scoped child token> }`
      - `env/ssh_key` = the private key tmpfile
      - `project/` = the cloned playbook
   9. `ansible_runner.run(...)`. Stream `stdout` lines to SignalR
      `RunLog_<run_id>` channel and append to local log file.
   10. On completion:
      - `vault list cloudmanager/vm/instances/{vm_pub}/{pb_pub}/` (recursive)
        → for each path, record an entry in `secrets_published`. Augment
        with labels/descriptions from `output_schema` if present.
      - Update `playbook_runs.{status, completed_at, stats, secrets_published}`.
      - Update `vm_playbook_assignments.{last_run_id, last_run_status, last_applied_at}`.
      - Revoke the scoped child token. Shred + delete the runner
        private_data_dir.
3. **Web UI** (subscribed to SignalR channel) sees stdout in real time;
   when status becomes terminal, refreshes the assignment view to show
   secrets.

## Secrets Strategy (Detail)

### Inputs the playbook needs (e.g. an API key for a downstream service)

- Mark the input in `argument_specs.yml` with `x-secret-path: <vault path>`
  (vendor extension — keys starting with `x-` are ignored by Ansible).
- Orchestrator resolves these using its own Vault token at apply time.
- Resolved values flow in as ordinary extra-vars.
- Playbook tasks that handle them set `no_log: true`.
- Web UI never asks the user to type a secret value for these — it shows
  the Vault path the value comes from.

### Outputs the playbook produces

- Playbook receives `secrets_prefix` (extra-var) and a scoped `VAULT_TOKEN`
  (env).
- Standard `community.hashi_vault.vault_write` against
  `{{ secrets_prefix }}/{{ relative_path }}` with `data: { value: "..." }`.
- After run, orchestrator lists the prefix and surfaces every key found.
- Web UI shows each secret as a row with a "Reveal" button (one-time
  `GET /api/v1/run/{runId}/secret/{key}` fetches from Vault, masks until
  click) and a "Copy path" button (always available).

## Worker Design

The C# API consumes Python (`ansible-runner` is Python-only) via a
**separate worker process** rather than embedding. Two reasons:

1. C#-to-Python interop is unpleasant. A subprocess is cleaner.
2. The worker can scale independently — multiple workers, RabbitMQ as the
   queue, idempotent runs.

**Recommended worker shape**: a small Python service (`cloud-manager-worker`),
new repo or new dir under cloud-manager-api/. Consumes RabbitMQ queue
`playbook-runs`. Each message is a `run_id`; the worker reads everything
else from the DB. Publishes status updates and stdout lines back to a
results queue that the C# API forwards to SignalR.

Reuse RabbitMQ infrastructure that's already up for vorch-service.

## UI

- **`/playbooks` page**: list registered playbooks, "Register new" modal
  with git_url + git_ref + playbook_path inputs.
- **`/playbook/{pid}` detail**: shows argument_schema fields (read-only
  documentation), output_schema, git_ref, last_refreshed.
- **`/virtualmachine/{vid}` page** gets a new "Playbooks" section:
  - Attached playbooks list, each with: name, last_run_status, last_applied_at, Apply / Edit / Detach buttons.
  - "Attach playbook" modal: pick from registered playbooks, parameter
    form auto-rendered from playbook.argument_schema via `@rjsf/core`.
  - Per-assignment expanded view: run history, latest run status, secrets
    published (collapsed by default, reveal-on-click).
- **`/run/{rid}` page**: live stdout (SignalR), status, stats, secrets.

## Phases

0. **Feature flag scaffolding** — `feature_flags` table, `/feature-flags`
   endpoint, `[RequireFeatureFlag]` attribute on controllers, web UI
   gating. Ship this first, with `playbooks.enabled=false` seeded. The
   rest of the work happens behind the flag.
1. **Schema only** — add tables, EntityPrefixRegistry, register-playbook
   endpoint that does git_ref clone, reads `meta/argument_specs.yml`,
   stores. No execution yet.
2. **Parameter form** — web UI renders argument_schema as a form; saves
   `vars_override` against assignments. Still no execution.
3. **Worker + ansible-runner** — Python worker, RabbitMQ queue, can
   actually run a playbook end-to-end against a VM. SSH key from Vault,
   stdout to logs. No Vault writes from playbook yet. Worker also runs
   the health probe and updates `feature_flags.playbooks.healthy`.
4. **Vault scoped tokens + outputs** — orchestrator mints child tokens,
   playbook writes to `secrets_prefix`, UI displays results.
5. **Run log streaming** — SignalR channel for live stdout.
6. **Schema diffing** — refresh + apply-time validation against current
   git_ref schema; warn user on drift.
7. **Flip the flag** — once Phases 1–6 are stable on at least one
   environment, `PATCH /admin/feature-flags/playbooks {enabled: true}`
   and the feature lights up.

## Open Questions

- **Worker placement**: separate `cloud-manager-worker` Python service vs.
  cohosting in vorch-service (which is Go and would need cgo or
  subprocess). Recommend separate service for clean dep boundaries.
- **Run log storage**: append-only `run_logs` table vs. file on
  cloud-storage. File is cheaper for large logs but loses SQL queryability.
  Recommend file with size-capped tail in DB for "last 200 lines" preview.
- **Playbook ordering / dependencies on a single VM**: do we need explicit
  "run playbook A before B"? Probably yes eventually (postgres before app).
  Deferred — Phase 1 of this work treats each apply as independent.
- **Multiple-host playbooks**: every playbook today targets one VM. Some
  playbooks (e.g. an HA cluster setup) need multiple. Defer — current
  schema scopes assignments per-VM.
- **Secret rotation**: when a user re-applies a postgres role, does the
  admin password rotate? Recommend the playbook decides (idempotent
  default = don't change; opt-in `force_rotate: true` extra-var). Out of
  scope here.

## Related Plans

- **`ANSIBLE-INSTALL-PLAN.md`** — idempotent install of ansible-runner,
  `community.hashi_vault`, hvac, and the Vault policy that lets the
  orchestrator mint scoped child tokens.
- **`PUBLIC-ID-PLAN.md`** (in `cloud-manager/planning/`) — public_id
  convention; `pb_*`, `asgn_*`, `run_*` follow that pattern.
