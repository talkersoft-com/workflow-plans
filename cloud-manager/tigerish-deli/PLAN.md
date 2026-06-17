# Plan: RabbitMQ Marketplace Blueprint for Cloud Manager

## Objective
Let a Cloud Manager user provision a brand-new VM with RabbitMQ already installed
and configured, in one click from the marketplace — exactly the way `postgres-jammy`
works today. We add a self-contained Ansible playbook that installs RabbitMQ on the
provisioned VM, then register a marketplace **Blueprint** (base VM image + that
playbook) so it appears as an installable tile and can be instantiated via the API.

## Background
PostgreSQL is already shipped as a marketplace item, and the machinery is fully
generic — adding a new service is *data*, not new code:

- **Playbooks** live as seed files in `cloud-manager-api/seed/playbooks/` as a
  `<name>.yaml` (Ansible play that runs on the VM) + `<name>.meta.json` (its
  argument/output JSON schema). Examples today: `pg-16-noble.*`, `pg-14-jammy.*`.
- `cloud-manager-mcp/.cicd/import-playbooks.py` syncs those seed files into the
  `vm.playbooks` table over the REST API (`POST/PATCH /api/v1/playbook`). It picks up
  any new files automatically — no edits needed.
- A **Blueprint** (`marketplace.blueprints`) bundles a `virtual_machine_image_id`
  with one or more playbooks via the `marketplace.blueprint_playbooks` join
  (position-ordered). Blueprints are seeded by an EF Core migration — see
  `20260611040351_AddMarketplaceBlueprints.cs`, which seeds `postgres-jammy` and
  links it to the `pg-14-jammy` playbook.
- The generic `MarketplaceController` / `IMarketplaceService` / `IPlaybookService`
  already list and instantiate any blueprint. The `marketplace` feature flag already
  exists. **No new controllers or services are required.**
- A local-dev installer (`scripts/rabbitmq/install-rabbitmq.py`) already exists, but
  that is **not** the on-VM Ansible playbook — that playbook does not exist yet and is
  the core new artifact.

## Design

### New artifact 1 — Ansible playbooks (run on the provisioned VM)
Two variants, mirroring Postgres' jammy + noble pair. Model `rabbitmq-jammy.yaml` on
`seed/playbooks/pg-14-jammy.yaml` and `rabbitmq-noble.yaml` on `pg-16-noble.yaml`.
The task bodies are identical except for the target release codename used in the apt
repo line (`jammy` vs `noble`).

`seed/playbooks/rabbitmq-jammy.yaml` (Ubuntu 22.04) and
`seed/playbooks/rabbitmq-noble.yaml` (Ubuntu 24.04)
- `become: true`; vars from the meta argument schema: `rabbitmq_user`, `rabbitmq_vhost`.
- Tasks:
  1. **Install from the official Team RabbitMQ apt repos** (chosen — gets current
     4.x, not the older distro package). Add the Team RabbitMQ signing keys and the
     two apt sources (modern Erlang repo + rabbitmq-server repo) for the matching
     release codename (`jammy`/`noble`), the same mechanism the pg playbook uses to add
     the PGDG repo, then apt install `erlang-base` deps + `rabbitmq-server`.
  2. Enable + start the `rabbitmq-server` service.
  3. Enable the `rabbitmq_management` plugin (`rabbitmq-plugins enable rabbitmq_management`).
  4. Generate a strong password; create the vhost `{{ rabbitmq_vhost }}`; create the
     admin user `{{ rabbitmq_user }}` with that password; set it as `administrator` and
     grant full permissions on the vhost. Remove the default `guest` user.
  5. Write the generated password to Vault — reuse the exact Vault task block from
     `pg-14-jammy.yaml` (same auth/path conventions), keyed for rabbitmq.
  6. Open firewall ports `5672` (AMQP) and `15672` (management UI).

`seed/playbooks/rabbitmq-jammy.meta.json` and `rabbitmq-noble.meta.json`
- `name`: `rabbitmq-jammy` / `rabbitmq-noble`
- `description`: `RabbitMQ 4.x on Ubuntu 22.04 LTS (Jammy)` /
  `RabbitMQ 4.x on Ubuntu 24.04 LTS (Noble)`
- `argumentSchema`: object with `rabbitmq_user` (string, default `admin`) and
  `rabbitmq_vhost` (string, default `/`).
- `outputSchema`: `password` (Vault path of the generated password) and
  `management_url`.

### New artifact 2 — EF Core migration (registers the marketplace Blueprint)
New migration in `cloud-manager-api/src/Models/CloudManager.Entities/Migrations/`,
modeled on `20260611040351_AddMarketplaceBlueprints.cs`.

`Up`: seed **two** blueprints (jammy + noble), each with its join row.
- `INSERT INTO marketplace.blueprints` × 2:
  - `name='rabbitmq-jammy'`, `description='RabbitMQ 4.x on Ubuntu 22.04 LTS (Jammy)'`,
    `virtual_machine_image_id = (SELECT id FROM vm.virtual_machine_images WHERE name='Ubuntu 22.04 LTS (Jammy)')`.
  - `name='rabbitmq-noble'`, `description='RabbitMQ 4.x on Ubuntu 24.04 LTS (Noble)'`,
    `virtual_machine_image_id = (SELECT id FROM vm.virtual_machine_images WHERE name='Ubuntu 24.04 LTS (Noble)')`.
  - Both: `default_memory=4096`, `default_disk_size_gb=20`, `status='Published'`.
- `INSERT INTO marketplace.blueprint_playbooks` × 2 — link each blueprint to the
  matching `vm.playbooks` row (`rabbitmq-jammy` / `rabbitmq-noble`), `position=0`,
  `vars_override='{}'`.
- Use the same idempotent `SELECT ... WHERE name=` joins the Postgres migration uses,
  so ordering vs. `import-playbooks.py` does not matter.

`Down`:
- Delete the `blueprint_playbooks` join rows, then the `blueprints` rows, for
  `name IN ('rabbitmq-jammy','rabbitmq-noble')`.

Do **not** recreate the `marketplace` feature flag — it already exists.

### Sequencing note
The blueprint→playbook link resolves by `vm.playbooks.name='rabbitmq-jammy'`, so the
playbook must be imported (via `import-playbooks.py`) for instantiation to resolve the
FK at runtime. Both orderings are safe because the seed INSERTs match by name; document
the run order in the deck (migrate, then import) for the verification step.

## Implementation phases

| Phase | Title              | Description                                                                                 |
|-------|--------------------|---------------------------------------------------------------------------------------------|
| 0000  | Setup              | `hv_status` + `hv_init`/`hv_next` to land on the execution branch                          |
| 0001  | RabbitMQ playbooks | Author `rabbitmq-jammy.*` (off `pg-14-jammy.*`) and `rabbitmq-noble.*` (off `pg-16-noble.*`), installing from the official Team RabbitMQ repos; verify YAML/schema |
| 0002  | Blueprint migration| Add the EF migration seeding both `marketplace.blueprints` rows + `blueprint_playbooks` links, with `Down`  |
| 0003  | Verify             | Run migration + `import-playbooks.py`; assert `GET /api/v1/marketplace` lists `rabbitmq-jammy` **and** `rabbitmq-noble`; instantiate one and confirm RabbitMQ reachable on 5672/15672 |

## Decisions (resolved)

- **Install source:** Official Team RabbitMQ apt repos (current 4.x) — not distro packages.
- **OS variants:** Ship both `rabbitmq-jammy` (22.04) and `rabbitmq-noble` (24.04),
  matching the Postgres pair.

## Open questions

- **Clustering:** Single-node only for v1? (Assumed yes — no clustering/HA in scope.)
- **Default sizing:** 4096 MB / 20 GB mirrors Postgres — confirm acceptable for RabbitMQ.
