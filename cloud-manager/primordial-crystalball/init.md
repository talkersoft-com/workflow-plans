# Cloud Manager — Secret Binding provisioning + injection (systemd creds via cloud-init)

## What this workflow does
Makes "VMs with systemd creds" real (workflow **#2**). At provision time, Cloud Manager resolves a
blueprint's attached Secret Bindings, reads the secret value(s) from Vault, and injects them into
the new VM as **systemd encrypted credentials** via **cloud-init user-data (NoCloud seed)**, sealed
**on first boot** with `systemd-creds encrypt` and then scrubbed. The VM is **never** a Vault
client — the controller reads Vault, the guest seals. On success, the `vm.vm_secret_bindings` usage
row is written (the delete-gate's lower joint).

## Read before starting
- `deck.md` — deck + repos in scope; the #1.5 dependency; the vorch manual redeploy note
- `Execution/Exec.md` — full task list and execution instructions
- `PLAN.md` — flow diagram, the 5 build items, phases, open questions
- `../../common/toolkit/hive-deck.md` — hv MCP call patterns

## Constraints
- **VMs are NEVER Vault clients.** The controller reads Vault; the guest only seals what it's given.
- **Secret values NEVER appear** in logs, run output, API responses, or MCP output. Use vorch's
  redact writer for any new log path. Values ride in `write_files` content (base64), **never** in
  `runcmd`/shell args (shell-injection guardrail, `provision.go:31`).
- **First-boot sealing only** — `systemd-creds encrypt` is host/TPM-bound; cannot be pre-encrypted
  on the controller. Plaintext lives only in guest **tmpfs** (`/run`) and is **shredded** after
  sealing.
- **Backward compatible contract:** a VM with no attached bindings provisions exactly as today
  (empty secrets list → no extra `write_files`/`runcmd`).
- **C# ↔ Go YAML lockstep** for the new `Secrets` payload — identical field/tag shape both sides.
- camelCase JSON on the API wire; public_ids only.
- **Depends on #1.5** (`Secret` entity, `secret_id`, empty `vm_secret_bindings`) being merged.
- **OUT OF SCOPE:** secret rotation/re-seal, TPM-bound sealing, SSH-push delivery, multi-VM
  fan-out, UI changes.
