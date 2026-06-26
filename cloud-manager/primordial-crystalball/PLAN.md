# Plan: Secret Binding provisioning + injection (systemd creds via cloud-init)

## Objective

Make "VMs with systemd creds" real. At VM provision time, Cloud Manager resolves a blueprint's
attached **Secret Bindings**, reads the secret value(s) from **Vault**, and injects them into the
new VM as **systemd encrypted credentials**, delivered through **cloud-init user-data (NoCloud
seed)** and sealed **on first boot** with `systemd-creds encrypt`. The VM is **never** a Vault
client — the controller reads Vault and pushes plaintext into the seed; the guest seals it and
scrubs the plaintext. When done, instantiating a blueprint that has a binding produces a VM whose
appliance unit reads its DB password (etc.) via `LoadCredentialEncrypted`, with no secret in any
log, API response, or persisted plaintext. This is workflow **#2** of the Secret Bindings feature
and **depends on #1.5** (the `Secret` entity + referential model + empty `vm_secret_bindings`)
being merged first.

## Background

- **#1 (shipped):** Secret Bindings authoring — bindings attach to blueprints
  (`marketplace.blueprint_secret_bindings`, with `cred_name`). No provision-time behavior.
- **#1.5 (planned/scaffolded):** the CM-owned `Secret` entity, nullable `secret_id` on bindings,
  the empty `vm.vm_secret_bindings` usage table, path-lock. #2 **populates** that usage table and
  **reads** `secret_id` / `vault_path`; it does not change that schema otherwise.
- **Verified current-state in code (the seams):**
  - **Vault read is MISSING.** `IVaultClient`
    (`cloud-manager-api/.../CloudManager.Vault.Client/IVaultClient.cs`) has only
    `WritePolicyAsync` / `DeletePolicyAsync` / `CreateChildTokenAsync` / `RevokeTokenAsync` — **no
    KV read**. The controller cannot fetch a secret value today. This is build item #1.
  - **`vm_public_id` is known before cloud-init** — `InitializeHost(PublicId, …)`
    (`vorch-lib/provision/provision.go:204`) validates it before rendering user-data, so templated
    `{vm_public_id}` paths can be resolved API-side before publish.
  - **cloud-init user-data is a hardcoded `fmt.Sprintf` template**
    (`vorch-lib/provision/provision.go:~313`): `write_files:` is netplan-only; `runcmd:` is
    resize/netplan/machine-id. Injection appends `write_files` + a first-boot seal step.
  - **The create-vm command contract has no secrets field.** C# `CreateVmCommandData →
    VmCommandData` (`cloud-manager-api/src/Models/CloudManager.Messages/`) ↔ Go `CreateVmMessage →
    CommandData{VM createvm.VM, Agents}` (`vorch-lib/models/messages/createvmmessage.go`,
    `vorch-lib/models/templates/createvm/`). A secrets payload must be added to **both** sides in
    YAML lockstep.
  - **Shell-injection guardrail** (`vorch-lib/provision/provision.go:31`): free-form values must
    not be interpolated into shell/virsh/paths. Secret **values** ride in `write_files` content
    (base64), never in `runcmd` args; the seal script reads the file.

## Design

### Flow (end to end)
```
instantiate blueprint
  └─ API: load attached secret_bindings for the blueprint
       └─ for each binding: compute concrete vault_path
            static   → Secret.vault_path (via secret_id)
            templated→ fill {vm_public_id} (+ params) from provision context
       └─ IVaultClient.ReadKvSecretAsync(path)  → secret value(s)
       └─ build resolved list: [{ credName, b64Value }]
  └─ publish CreateVmCommandData (now carrying secrets[])  ── RabbitMQ ──▶ vorch
       └─ InitializeHost renders cloud-init user-data:
            write_files: /run/cm-seed/<credName> ← b64 value (0600 root)
            runcmd: first-boot → systemd-creds encrypt --with-key=host
                      /run/cm-seed/<credName> → /etc/credstore.encrypted/<credName>
                    then shred the /run/cm-seed plaintext
       └─ guest appliance unit: LoadCredentialEncrypted=<credName>:/etc/credstore.encrypted/<credName>
  └─ on provision success: INSERT vm.vm_secret_bindings { vm_id, secret_binding_id, cred_name }
```

### 1. Vault KV read (cloud-manager-api)
Add `Task<IReadOnlyDictionary<string,string>> ReadKvSecretAsync(string path)` to `IVaultClient` +
`VaultClient` — KV v2 read against the `cloudmanager` mount, returning the secret's fields. The
value is **never logged**. Reuses the existing controller Vault token configuration.

### 2. Binding resolution at instantiate (cloud-manager-api)
In the instantiate path (`MarketplaceController` → `ProvisionService` / `BlueprintRunSequencer`):
- load the blueprint's attached `blueprint_secret_bindings` (with `cred_name`, position),
- for each binding compute the concrete path:
  - **static** (`secret_id` set) → the referenced `Secret.vault_path`,
  - **templated** (`secret_id NULL`) → substitute `{vm_public_id}` (and any other params) from the
    provision context into `path_template`,
- `ReadKvSecretAsync` each → assemble `[{ credName, b64Value }]` (base64 of the chosen field;
  default field convention documented — see Open Questions),
- attach to the create-vm command. **Empty list when no bindings** → behavior identical to today.

### 3. Create-vm contract extension (C# + Go, YAML lockstep)
- C#: add `List<SecretCredData> Secrets` to `VmCommandData` (`{ CredName, B64Value }`).
- Go: add the matching field to `createvm.VM` / `CommandData` with identical YAML tags.
- Base64 for safe YAML transport; **not** pre-encrypted (sealing is host-bound, must happen in the
  guest). Backward compatible: omitted/empty deserializes to no secrets.

### 4. Cloud-init injection + first-boot seal (vorch-lib / vorch-service)
Extend `InitializeHost` (new param: resolved secrets) and the user-data builder so that, for each
secret:
- a `write_files` entry writes the **base64** value to `/run/cm-seed/<credName>` (mode `0600`,
  root) — `/run` is tmpfs, so it never persists on the guest disk,
- a `runcmd`/`bootcmd` first-boot step: `base64 -d` → `systemd-creds encrypt --with-key=host
  --name=<credName> - /etc/credstore.encrypted/<credName>` (dir `0700`), then `shred -u` the
  `/run/cm-seed/<credName>` plaintext.
- `credName` is validated against a strict charset (mirror the publicId guardrail) so it can be
  safely used in paths; values never touch `runcmd` args.

### 5. Record VM↔binding usage (cloud-manager-api)
On successful provision, insert `vm.vm_secret_bindings { virtual_machine_id, secret_binding_id,
cred_name }` per injected binding — the delete-gate's lower joint (RESTRICT) from #1.5. (Idempotent
on retry.)

### Decisions taken (defaults — override in review)
- **Delivery:** extend the existing **NoCloud user-data** path; not SSH-push.
- **Plaintext lifecycle:** plaintext lives only in tmpfs (`/run`) in the seed, **shredded** right
  after sealing. Documented residual exposure: the host-side NoCloud **seed image** contains the
  plaintext until first boot completes.
- **Sealing key:** `systemd-creds encrypt --with-key=host` (TPM-bound deferred).
- **cred target:** `/etc/credstore.encrypted/<credName>`; the consuming unit
  (`LoadCredentialEncrypted`) is **baked into the marketplace appliance image**.

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000 | Setup | hv_status + hv_init/hv_next; record branch |
| 0001 | API — Vault KV read | `ReadKvSecretAsync` on IVaultClient/VaultClient (KV v2, cloudmanager), value never logged; unit-tested against the live Vault path |
| 0002 | API — resolve bindings at instantiate | load attached bindings → compute static/templated path → read Vault → build resolved `[{credName,b64Value}]`; empty when no bindings |
| 0003 | Contract — create-vm secrets payload (C# + Go) | add `Secrets` to VmCommandData + createvm.VM in YAML lockstep; round-trip a message both directions; backward compatible |
| 0004 | vorch — cloud-init injection + first-boot seal | write_files (tmpfs, 0600) + runcmd `systemd-creds encrypt --with-key=host` + shred; credName charset validation; no value in logs |
| 0005 | API — populate vm_secret_bindings | insert usage rows on provision success (idempotent); proves the delete-gate lower joint |
| 0006 | E2E + build + deploy | provision a VM from a blueprint with a binding → confirm `/etc/credstore.encrypted/<credName>` exists, plaintext gone, appliance reads cred; no-binding VM unchanged; build/deploy api + (vorch redeploy); write Results + LESSONS; hv_integrate |

## Open questions

- **Which Vault field becomes the credential?** A KV secret has multiple fields (e.g. postgres
  writes `{host,port,db,user,password}`). Default assumption: the binding injects a **single
  named field** (e.g. `password`) per `cred_name`. Confirm whether a binding should be able to map
  several fields → several creds in one shot, or inject the whole JSON blob as one cred.
- **Seed at-rest plaintext** — accepted as transient (shredded post-seal, tmpfs in guest), but the
  **host NoCloud seed image** holds plaintext until first boot. Acceptable, or should vorch delete
  the seed artifact after the domain reports first-boot completion?
- **vorch deploy path** — vorch-service runs on the hypervisor host; confirm the redeploy/restart
  step for vorch is part of this workflow's deploy (the cloud-manager deploy script targets
  api/mcp/web, not vorch). May need a documented manual vorch redeploy in deck.md.
- **Templated params beyond `{vm_public_id}`** — today only `vm` source exists. Confirm no other
  param source is needed at instantiate for the first cut.
