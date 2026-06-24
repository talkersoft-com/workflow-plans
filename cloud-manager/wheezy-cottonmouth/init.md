# Cloud Manager — Secret Manager (CM-owned secrets + referential integrity)

## What this workflow does
Adds a first-class, **Cloud-Manager-owned `Secret`** entity and a **Secret Manager** utility so
an operator can create / list / delete Vault secrets manually. Creating a secret writes it to
Vault under a CM-composed `cloudmanager/data/manual/{slug}` path **and** records a DB row that it
exists. A **3-level referential-integrity delete gate** then protects the chain: you cannot
delete a `Secret` while a `SecretBinding` references it, nor a `SecretBinding` while a VM uses it;
only at the leaf (a Secret with zero live bindings) does delete also hard-delete the value from
Vault. This is workflow **#1.5** — authoring shipped (#1); provisioning/injection is the later #2.

## Read before starting
- `deck.md` — deck + repos in scope; pre-written hv MCP calls
- `Execution/Exec.md` — full task list and execution instructions
- `../../common/toolkit/hive-deck.md` — hv MCP call patterns
- `PLAN.md` — objective, data model, the 3-level delete gate, the create/delete state machine, phases

## Constraints
- **Path-lock is non-negotiable:** every secret path is under the `cloudmanager` mount
  (`cloudmanager/data/…`), composed by CM, never free-form typed. Enforce in **three layers** —
  DB `CHECK (vault_path LIKE 'cloudmanager/data/%')`, API validation, and a UI fixed-prefix
  composer (no free-form path field).
- **3-level delete gate:** `vm_secret_bindings.secret_binding_id` and `secret_bindings.secret_id`
  are both **ON DELETE RESTRICT**. Leaf delete (Secret, 0 live bindings) removes the DB row AND
  hard-deletes from Vault.
- **Strict, DB-first state machine** for create/delete (`Pending`/`Active`/`Failed`/`Deleting`/
  `Deleted`) so there is never a Vault key the DB doesn't know about.
- **`secret_id` is nullable** — every Secret Binding shipped in #1 stays untouched (templated,
  `secret_id = NULL`). Backward compatibility is mandatory.
- camelCase JSON on the wire; **public_ids only** (never leak Guids); **soft-delete authoritative**.
- The migration is **reversible** (clean up/down), applied BEFORE the api restart on deploy.
- VMs are **never Vault clients** — the controller's Vault token does all Vault I/O.
- Mirror the existing `secret_bindings` code paths — do not invent a parallel paradigm.
- **OUT OF SCOPE:** provisioned-secret registration (`origin=provisioned`), instantiate-time
  resolution, reading Vault at provision, cloud-init/systemd-creds injection, create-vm changes,
  and **populating `vm_secret_bindings`** (the table lands empty now only to anchor the FK).
