# Plan: Secret Bindings — authoring foundation

## Objective

Add **Secret Bindings** to Cloud Manager: a reusable, parameterized definition of a Vault
secret that can be **attached to marketplace blueprints** — the secret-world parallel to how
Playbooks attach to blueprints. This workflow delivers the **authoring half**: define Secret
Bindings and attach them to blueprints, across API + MCP + web. When done, an operator can
create the two canonical bindings (a no-param "manual" one and a `{vm_public_id}` "postgres"
one) in the UI, attach them to a blueprint, and round-trip them via the API and MCP.
**Provisioning-time resolution and secret injection are a separate follow-up workflow** — but
the schema lands their extension points now so the follow-up needs no migration churn.

## Background

- **Marketplace blueprints** today wire a VM image + playbooks (`marketplace.blueprints` +
  `marketplace.blueprint_playbooks` join, position-ordered, `vars_override jsonb`), published
  to the marketplace and instantiated by `MarketplaceController`. Secret Bindings mirror this
  exact shape — a reusable entity + a per-blueprint join.
- **Secrets live in Vault** under the `cloudmanager` mount, e.g. a provisioned VM's creds at
  `cloudmanager/data/vm/instances/{vm_public_id}/postgres` (the playbook writes them; the
  controller token reads them). A Secret Binding's `path_template` points at such a path.
- **The web app already uses `@rjsf`** (react-jsonschema-form) to render playbook
  `argumentSchema` forms — reuse it for the params editor and (later) the instantiate form.
- **`vm_public_id`** is the canonical VM identifier (Stripe-style `vm_…`), used in Vault paths.
- This is workflow **#1 of the Secret Bindings feature**; #2 is provisioning (resolve params at
  instantiate + inject via cloud-init systemd-creds).

## Design

### The core mechanic — `params` is the single indicator
A Secret Binding's behavior is driven entirely by its `params` list:
- **empty `params`** → no inputs needed (the "manual" pattern, `/manual/secret/i/stored`).
- **a param with `source: "vm"`** → at provision time (follow-up workflow) the user picks a VM;
  the chosen `vm_public_id` fills the `{vm_public_id}` placeholder.

`params` is derived from the `{placeholders}` in `path_template`. No extra flags.

### Entity — `SecretBinding` (schema: `marketplace`)
| Column | Type | Notes |
|---|---|---|
| `id` | uuid | internal, never exposed |
| `public_id` | varchar(32) | `sb_…` |
| `name`, `description` | varchar | |
| `path_template` | text | e.g. `cloudmanager/data/vm/instances/{vm_public_id}/postgres` |
| `params` | jsonb | array of `{ name, source, label }`; `[]` = no params. `source` enum (extensible), today only `"vm"` |
| `injection_provider` | varchar(32) | extensible enum, default `"systemd-creds"` — **stored now, used by workflow #2** |
| `status` | varchar(32) | `Draft` / `Published` |
| `config` | jsonb | future-extensibility bag |
| `deleted_at` | timestamp | soft-delete (authoritative) |

### Join — `blueprint_secret_bindings` (parallel to `blueprint_playbooks`)
| Column | Type | Notes |
|---|---|---|
| `id` / `public_id` | uuid / `bsb_…` | |
| `blueprint_id` | uuid FK | cascade delete with blueprint |
| `secret_binding_id` | uuid FK | |
| `position` | int | unique per blueprint |
| `cred_name` | varchar | the systemd credential name on the target (e.g. `DBPASSWD`) — **stored now, used by workflow #2** |

### cloud-manager-api (.NET)
- EF entities `SecretBinding` + `BlueprintSecretBinding`; **one reversible migration**
  creating both tables in the `marketplace` schema.
- CRUD endpoints for Secret Bindings: list / get / create / update / soft-delete.
- Blueprint↔binding: attach / detach / list / reorder.
- Conventions: **camelCase JSON, public_ids only (never leak Guids), soft-delete authoritative**,
  mirror the `blueprint_playbooks` service/controller code paths. **No instantiate changes.**

### cloud-manager-mcp (TS)
Tools mirroring the API: `cloud_secret_binding_list/get/create/update/delete` and
`cloud_blueprint_secret_binding_attach/detach/list`.

### cloud-manager-web (React/Vite/@rjsf)
- A **"Secret Bindings"** management page: list + create/edit form (name, description,
  `path_template`, params editor). As the author types `path_template`, **scan for
  `{placeholders}`** and render one param row each with a **source picker** (only `"vm"` today;
  a placeholder named `vm_public_id` defaults to `source: vm`).
- On the **Blueprint detail page**, an **"Attached Secrets"** section to attach / detach /
  reorder Secret Bindings — parallel to the existing playbook-attachment UI.
- **No instantiate-form changes** in this workflow.

### Extension points (designed now, built in workflow #2)
- `params[].source` — today `"vm"`; tomorrow `"network"`, `"static"`, … (one resolver + one widget each).
- `injection_provider` — today `"systemd-creds"`; tomorrow `"ssh-push"`, `"vault-agent"`, … (one provider each).
- `cred_name` on the join — consumed by the injection provider at provision time.

## Implementation phases

| Phase | Title | Description | Test |
|-------|-------|-------------|------|
| 0000 | Setup | branch via hv | clean status |
| 0001 | API entities + migration + endpoints | `SecretBinding` + `BlueprintSecretBinding` entities; one reversible migration; CRUD + attach/detach/reorder/list endpoints | migration up/down clean; CRUD round-trips both example bindings; camelCase + public_ids; soft-delete hides |
| 0002 | MCP tools | `cloud_secret_binding_*` + `cloud_blueprint_secret_binding_*` mirroring the API | tools create/list/attach the two examples against the live API |
| 0003 | Web — Secret Bindings page + blueprint Attached Secrets | management page + params editor (placeholder scan + source picker); blueprint "Attached Secrets" attach/detach/reorder | author both examples in the UI; attach one to a blueprint; reorder persists |
| 0004 | Verify + build + deploy | author both examples (manual no-param + postgres vm-param), attach, round-trip via API/MCP/UI; build api+mcp+web; migration applied before api restart; deploy api+web | end-to-end round-trip green; services healthy |

## Open questions

*(Terminology + scope locked: "Secret Binding"; authoring only; provisioning is workflow #2.)*
- **Reusable vs blueprint-scoped:** confirm Secret Bindings are a **reusable library** (one binding attachable to many blueprints) — assumed yes (mirrors playbooks). If they should be blueprint-private, the join collapses into the binding row.
- **`status` (Draft/Published):** do Secret Bindings need a publish gate of their own, or is the blueprint's publish state sufficient? (Default: include the column, don't gate on it yet.)
