# Plan: Secret Manager — CM-owned secrets + referential-integrity model

## Objective

Give Cloud Manager a first-class, **Cloud-Manager-owned `Secret`** entity and a **Secret
Manager** utility to create / list / delete Vault secrets manually, with a **3-level
referential-integrity delete gate** across `Secret → Binding → VM`. Today a Secret Binding
carries a free-form `path_template` — an operator can point it at any Vault path, including
ones Cloud Manager does not own. This workflow closes that leak: every CM secret lives under
the **`cloudmanager` mount** at a CM-composed path (never free-form typed), CM records that
the secret exists in its own database, and the database enforces that you cannot delete a
secret while a binding references it, nor a binding while a VM uses it. When done, an operator
can create a manual secret in the UI (it lands in Vault **and** as a DB row), reference it
from a Secret Binding, and observe the delete gate at each level. This is workflow **#1.5** of
the Secret Bindings feature — authoring shipped (#1); provisioning/injection is the later #2.

## Background

- **Secret Bindings authoring shipped (#1):** `marketplace.secret_bindings` (public_id `sb_`,
  free-form `path_template`, `params` jsonb, `injection_provider`, `status`, soft-delete) +
  `marketplace.blueprint_secret_bindings` join (`cred_name`). The authoring page renders
  `path_template` as a **free-form text input** — nothing constrains it to a CM-owned path,
  and the shipped canonical example (`/manual/secret/i/stored`) is itself un-owned. That is the
  leak this workflow fixes.
- **Secrets live in Vault** under the `cloudmanager` mount (KV v2), e.g. a provisioned VM's
  creds at `cloudmanager/data/vm/instances/{vm_public_id}/postgres`. The controller's Vault
  token reads/writes them; **VMs are never Vault clients**.
- **`vm_public_id`** is the canonical VM identifier (Stripe-style `vm_…`).
- The user's model, verbatim intent: *create the secret → it goes in Vault AND a DB row records
  it exists → a binding references it (now the secret can't be deleted) → a VM uses the binding
  (now the binding can't be deleted)*. Deletion is a **3-level gate**, and the destructive Vault
  delete only happens at the leaf, when nothing references the secret.
- This reshapes the data model but is **backward compatible**: the new `secret_id` FK on
  `secret_bindings` is nullable, so every binding that shipped in #1 keeps working untouched.

## Design

### New entity — `Secret` (schema: `marketplace`)
| Column | Type | Notes |
|---|---|---|
| `id` | uuid | internal, never exposed |
| `public_id` | varchar(32) | `sec_…` |
| `name`, `description` | varchar | |
| `vault_path` | text | **`CHECK (vault_path LIKE 'cloudmanager/data/%')`** — CM owns the namespace; composed by CM, never free-form typed |
| `origin` | varchar(32) | `manual` (Secret Manager) today; `provisioned` reserved for #2 — **not registered in this workflow** |
| `status` | varchar(32) | state machine: `Pending` / `Active` / `Failed` / `Deleting` / `Deleted` |
| `config` | jsonb | future-extensibility bag |
| `deleted_at` | timestamp | soft-delete (authoritative) |

### Altered — `secret_bindings`
Add **`secret_id` uuid NULL FK → `marketplace.secrets(id)` ON DELETE RESTRICT**.
- **Static binding** (manual secret): `secret_id` set; `path_template` derived from the Secret (read-only in UI).
- **Templated binding** (the `{vm_public_id}` postgres case): `secret_id = NULL` + `path_template` pattern — unchanged from #1.
- Every binding shipped in #1 is the templated shape (`secret_id = NULL`) and is untouched.

### New join — `vm.vm_secret_bindings` (lands now, **populated in #2 only**)
| Column | Type | Notes |
|---|---|---|
| `id` / `public_id` | uuid / `vsb_…` | |
| `virtual_machine_id` | uuid FK | the VM that uses the binding |
| `secret_binding_id` | uuid FK | **ON DELETE RESTRICT** — this is the gate's lower joint |
| `cred_name` | varchar | the systemd credential name injected on the VM |
| `created_at` | timestamp | |

Created empty now purely to anchor the integrity FK; the controller writes rows in workflow #2.

### The 3-level delete gate (referential integrity)
```
   VM ──uses──▶ Binding ──references──▶ Secret ──stored in──▶ Vault
   1. can't delete Binding while a VM uses it    → vm_secret_bindings.secret_binding_id RESTRICT
   2. can't delete Secret  while a Binding refs it → secret_bindings.secret_id          RESTRICT
   3. leaf delete (Secret, 0 live bindings) → remove DB row AND hard-delete from Vault
```
The Vault hard-delete is safe **because it can only fire at the leaf** — nothing references the
secret by the time it happens.

### Secret create/delete state machine (strict, DB-first → never an unknown Vault orphan)
- **Create:** `INSERT … status=Pending` → write value to Vault at the locked
  `cloudmanager/data/manual/{slug}` path → `UPDATE status=Active`. Vault write fails →
  `status=Failed` (retryable; the DB always knows about any Vault key).
- **Delete:** guard (0 live bindings, else **409**) → `status=Deleting` → delete from Vault →
  `status=Deleted` (`deleted_at` set, kept for audit). Vault delete fails → stays `Deleting`
  (retryable). A reconciler can always converge: every Vault key has a DB row stating its intent.

### Path-lock — three layers, single source of truth = the `cloudmanager` mount + `data/`
1. **DB** `CHECK (vault_path LIKE 'cloudmanager/data/%')` — the backstop nothing bypasses.
2. **API** validation — reject any path not under the owned prefix (also normalizes the slug).
3. **UI** — no free-form path field; the `cloudmanager/data/manual/` prefix is a **fixed,
   non-editable** label and the operator only supplies a slug.

### cloud-manager-api (.NET)
- EF `Secret` + `VmSecretBinding` entities; `SecretBinding` gains the nullable `secret_id`.
- **One reversible migration** (clean up/down) creating `marketplace.secrets`,
  `vm.vm_secret_bindings`, and the `secret_bindings.secret_id` column + both RESTRICT FKs +
  the `vault_path` CHECK.
- Secret endpoints: list / get / create (Vault write + DB via the state machine) /
  delete (guarded; Vault hard-delete at the leaf). Vault access reuses the existing controller
  Vault token path.
- Conventions: **camelCase JSON, public_ids only (never leak Guids), soft-delete authoritative**;
  mirror the existing `secret_bindings` service/controller code paths.

### cloud-manager-mcp (TS)
- Tools mirroring the API: `cloud_secret_list / get / create / delete`.

### cloud-manager-web (React/Vite)
- A **Secret Manager** page: list (name, path, status badge), create form with the fixed
  `cloudmanager/data/manual/` prefix + slug input + value field(s) (no free-form path),
  delete with the guard surfaced (clear 409 message when a binding still references it).
- On the Secret Bindings authoring page: allow a binding to reference an existing managed
  Secret (static binding) in addition to the templated path; keep templated authoring working.

### Out of scope (deferred to workflow #2)
Provisioned-secret registration (`origin=provisioned` rows), instantiate-time param resolution,
reading Vault at provision, cloud-init/systemd-creds injection, create-vm changes, and
**populating `vm_secret_bindings`**. That table is created empty now only to anchor the FK.

## Implementation phases

Each phase maps to one TASK file during execution. Keep phases independent and testable.

| Phase | Title | Description | Test |
|-------|-------|-------------|------|
| 0000  | Setup | branch via hv (already on `wheezy-cottonmouth`) | clean status |
| 0001  | API — entities, migration, Secret CRUD + state machine | `Secret` + `VmSecretBinding` entities; nullable `secret_id` on `SecretBinding`; one reversible migration (secrets + vm_secret_bindings + secret_id FK + vault_path CHECK); create (Vault write + Pending→Active) / delete (guard→Deleting→Vault delete→Deleted) / list / get | migration up/down clean; create lands in Vault + DB Active; delete blocked (409) while a binding refs the secret; leaf delete removes Vault key + row; path outside `cloudmanager/data/` rejected by API and CHECK |
| 0002  | MCP tools | `cloud_secret_list / get / create / delete` mirroring the API | tools create a manual secret, list it, attach a binding, observe delete-guard against the live API |
| 0003  | Web — Secret Manager page + binding→secret reference | Secret Manager page (list, fixed-prefix create, guarded delete); Secret Bindings page gains "reference managed Secret" (static binding) while keeping templated authoring | author a manual secret in the UI (lands in Vault+DB); reference it from a binding; delete is blocked with a clear message; templated binding still authorable |
| 0004  | Verify + build + deploy | end-to-end: create manual secret → reference from binding → delete-gate at each level; build api+mcp+web; migration applied before api restart; deploy api+web | round-trip green; gate enforced at both joints; services healthy |

## Open questions

*(Both prior open decisions are resolved: leaf-delete hard-deletes from Vault, gated by the
3-level chain; create/delete use the strict DB-first state machine.)*

- **`vm_secret_bindings` schema placement** — assumed `vm` schema (alongside VM tables). If
  VM-related marketplace joins live under `marketplace`, place it there instead. Either way the
  RESTRICT FK to `secret_bindings` is what matters.
- **Slug derivation** — assumed the operator supplies a slug and the API normalizes it
  (lowercase, `[a-z0-9-]`), composing `cloudmanager/data/manual/{slug}`; uniqueness enforced on
  `vault_path`. Confirm whether the display `name` should auto-suggest the slug.
