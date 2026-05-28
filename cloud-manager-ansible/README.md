# cloud-manager-ansible — implementation planning bundle

## Problem statement

The ansible data model is fully deployed in `clouddb` (12 ansible-related tables in the `vm` schema, plus `membership.feature_flags`), and Entity Framework migrations through `AddPlaybookManagement` are applied. But the surfaces that make those tables useful — the REST API for editing playbooks, roles, and role files; the web UI for the operator; the Go event processor that actually runs playbooks against VMs — do not exist yet. Today an operator can only manage roles by direct SQL, runs queue but never execute (the publisher fires but no consumer subscribes), and the playbook content field is unreachable from any client. This bundle is the cross-project design for closing that gap.

## Bundle contents

- **[API-PLAN.md](./API-PLAN.md)** — REST endpoints, DTOs, services, controllers, and revision-management hooks for the cloud-manager-api .NET project. Covers every ansible entity that currently has no API surface plus the materialization endpoint the worker consumes.
- **[WEB-PLAN.md](./WEB-PLAN.md)** — A dedicated, self-contained Ansible section in cloud-manager-web. Clean separation from VM management: `VmPlaybooks` deleted, ansible lives only under `/ansible/*`. Also adds a `SettingsPage` for in-UI feature-flag control.
- **[VORCH-PLAN.md](./VORCH-PLAN.md)** — Go event processor inside vorch-service that consumes the existing `playbook-runs` queue, materializes playbook trees from DB content, execs `ansible-runner` as a subprocess, and manages two distinct Vault tokens per run (worker's long-lived token + per-run scoped child token for the playbook).
- **[DATA-MODEL-DELTAS.md](./DATA-MODEL-DELTAS.md)** — Audit of column / index / constraint requirements that the three plans above surface against the existing schema. Either a concrete migration shopping list or an explicit "no changes required" with per-plan justification.

## What's already shipped

The database side is done and verified live:

- **Tables** (12 ansible-related in `vm` schema): `playbooks`, `playbook_revisions`, `ansible_roles`, `role_files`, `role_file_revisions`, `playbook_global_role_refs`, `playbook_roles`, `playbook_role_files`, `playbook_role_file_revisions`, `vm_playbook_assignments`, `playbook_runs`, `playbook_run_targets`. Plus `membership.feature_flags`.
- **Migrations applied**: 6 migrations through `AddPlaybookManagement` (which dropped the git-backed fields and reshaped `PlaybookRun` to own execution via `playbook_id` instead of `assignment_id`).
- **EntityPrefixRegistry** entries: `pb`, `pbrev`, `ansr`, `rf`, `rfrev`, `pgr`, `plr`, `plrf`, `plrfr`, `prt`, `asgn`, `run`, `ff`. No new prefixes needed unless `DATA-MODEL-DELTAS.md` justifies them.
- **Helpers**: `ArgumentSpecsTranslator` and `SchemaDiff` exist at `cloud-manager-api/src/Services/CloudManager.Data.Services/Playbooks/` — currently unused, ready to be wired in by the API plan.
- **Publisher**: `IPlaybookRunPublisher` is wired into `RunController.Apply` — runs already enqueue to RabbitMQ. The consumer is the gap `VORCH-PLAN.md` fills.

See [`../DATA-MODEL.md`](../DATA-MODEL.md) for the full entity-relationship diagram and identifier strategy.

## Relationship to existing plans

Two earlier planning files live alongside this bundle in `cloud-manager/planning/planning/`:

- **[`../ANSIBLE-INSTALL-PLAN.md`](../ANSIBLE-INSTALL-PLAN.md)** — fully authoritative. Specifies the idempotent install of `ansible-runner`, the `community.hashi_vault` collection, `hvac`, and the Vault policy (`cloudmanager-playbook-runner`) plus token role (`playbook-run`) that the worker mints child tokens from. `VORCH-PLAN.md` assumes this install is complete on the worker host.
- **[`../PLAYBOOK-INTEGRATION-PLAN.md`](../PLAYBOOK-INTEGRATION-PLAN.md)** — partially stale. Authoritative sections that this bundle preserves:
  - Feature-flag gating model (three states: disabled, enabled-but-unhealthy, enabled-and-healthy) and the `[RequireFeatureFlag("playbooks")]` attribute.
  - Predictable Vault path pattern (`cloudmanager/vm/instances/{vm_pub}/{playbook_pub}/<output_path>`).
  - Secrets strategy (input resolution via `x-secret-path`, output convention via `meta/outputs.yml`).
  - Run-log storage trade-offs (file on disk for size, optional tail in DB for preview).
  - Health-probe semantics distinguishing "enabled but broken" from "enabled and working".

  Superseded sections (use this bundle instead):
  - **Data Model Additions** — describes the old git-backed `playbooks` table shape (`git_url`, `git_ref`, `playbook_path`, `last_refreshed_at`) and ties `PlaybookRun` to `assignment_id`. The deployed schema is different — see `../DATA-MODEL.md` for current truth.
  - **Worker Design** — recommends a Python worker (`cloud-manager-worker`). `VORCH-PLAN.md` replaces this with a Go-in-vorch-service design and dispositions the empty Python scaffold (delete after VORCH-PLAN Phase 1 lands).
  - **API Surface listing** — incomplete relative to the entities now in the schema. Use `API-PLAN.md`.
  - **UI section** — predates the clean-separation decision. Use `WEB-PLAN.md`.

## Implementation order

These plans are independent enough to ship on parallel branches once `DATA-MODEL-DELTAS.md` lands (if anything is needed). Recommended sequence:

1. **`DATA-MODEL-DELTAS.md` first** — if it lists migrations, ship them before any of the surfaces depend on them. If it concludes "no changes required", skip and proceed.
2. **`API-PLAN.md` next** — the worker and the web both consume API endpoints; without them neither can be built end-to-end. The API also has the most internal layering (entity → service → DTO → controller → tests), so it benefits from going first.
3. **`WEB-PLAN.md` and `VORCH-PLAN.md` in parallel** — they share no code. Web depends only on the API; vorch depends on the API's materialization endpoint and the existing RabbitMQ publisher. Each will be its own follow-on workflow with its own scaffold and bundle.

Each plan ends with a `## Phases` section describing the smallest shippable slice first; the implementation workflows should follow those phases as their own task list. This bundle stops at design — implementation lives in three follow-on workflows.
