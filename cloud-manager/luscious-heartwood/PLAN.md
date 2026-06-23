# Plan: Debian packaging + secret-injection for the Cloud Manager controller

## Objective

Replace the ad-hoc `scp` + per-service install scripts with reproducible **Debian
packages** that both **install from scratch** and **upgrade/adopt** an existing
Cloud Manager controller on Ubuntu 24.04 — with secrets injected via **systemd
encrypted credentials** instead of plaintext, and a **periodic, auto-renewing
Vault token** so the silent-expiry outage that motivated this work cannot recur.
End state: `apt install cloud-manager` stands up a controller; the same package
run on the existing box adopts it non-destructively; no secret is ever in a unit
file or plaintext env file; the token renews itself on a timer; and a fresh
install in a throwaway VM yields a working VM-provisioning stack.

## Background

- **Motivating incident:** the long-lived static Vault token in `/etc/cloud-manager/api.env`
  expired/was-revoked. Every service shares it, so `vorch_service` got 403 writing
  per-VM SSH keys to `cloudmanager/data/vm/instances/{id}/ssh`; VM provisioning failed
  silently (`orchestrationStatus: 3`) with no alert. **Already repaired live**: a 1-year
  **periodic** `cloudmanager` token now runs (after raising `auth/token` `max_lease_ttl`
  to `8760h`), verified by a successful provision. This plan makes that durable + repeatable.
- **Secret sprawl (confirmed on the box):** the Vault token lived in **three** places
  (`/etc/cloud-manager/api.env`, `/kvm-automator/config-server.yaml`, `porch.service`).
  The `cloud-manager-api` unit also carries `DBPASSWD=P@ssw0rd!` and `RABBITMQ_PASSWORD=P@ssw0rd!`
  in cleartext `Environment=` lines (readable via `systemctl cat`).
- **Platform:** Ubuntu 24.04, systemd 255, `systemd-creds` present, TPM2 = partial →
  `LoadCredentialEncrypted` available (host/TPM-bound, tmpfs, per-service, 0400).
- **Components** (all on the one controller; PG + RabbitMQ on localhost):
  `cloud-manager-api` (.NET, :5250), `vorch_service` (Go, root, libvirt/KVM),
  `porch` (Go, ansible-runner), `cloud-manager-web` (static React/Vite SPA, served by `serve`).
- **Auth model (confirmed in code):** API (.NET `VaultClient`) + vorch/porch (Go vault
  client) read a static token. The **Go client already supports `VAULT_TOKEN_FILE`**;
  the .NET client does not yet.
- **RabbitMQ creds have three consumers** — API, vorch, porch — so rotating that password
  is a three-touch-point change (the same shape as the token incident).

## Design

### Packages
1. **`cloud-manager-controller`** — `cloud-manager-api` + `vorch_service` + `porch`
   binaries, systemd units, non-secret config. `Depends:` libvirt/qemu, ansible-runner,
   postgresql, rabbitmq-server. Lay files/units out so **vorch can later split into
   `cloud-manager-vorch`** (per-hypervisor) without rework.
2. **`cloud-manager-web`** — static bundle served by **nginx** (drop the Node `serve`
   dependency); nginx reverse-proxies `/api` → `:5250`. No secrets.
3. **`cloud-manager`** — metapackage, `Depends:` the other two.

### Secret loading — fixed names, flexible source
- **Canonical key names are the contract:** `VAULT_TOKEN`, `DBPASSWD`, `RABBITMQ_PASSWORD`.
- **systemd credentials (files, not env vars):** install seals each secret —
  `systemd-creds encrypt` → `/etc/cloud-manager/creds/<name>.cred` (host/TPM2-bound).
  Units gain `LoadCredentialEncrypted=<name>:/etc/cloud-manager/creds/<name>.cred`;
  systemd decrypts to `$CREDENTIALS_DIRECTORY/<name>` (tmpfs, 0400, per-service).
- **Consumption:**
  - vorch/porch (Go): `Environment=VAULT_TOKEN_FILE=%d/vault-token` — already supported, no code change.
  - API (.NET): add the **`AddKeyPerFile($CREDENTIALS_DIRECTORY)`** config provider, **layered**
    `env → cred → Vault` (highest present source wins). The cred file named `DBPASSWD`
    becomes config key `DBPASSWD`, so existing API code keeps working; it just sources the
    value from the decrypted file instead of the environment. Backward-compatible (env still
    works where no cred is present). No per-operator mapping file — operators only choose
    the `.cred` path in the unit.
- Remove all plaintext `Environment=<secret>` lines from the units.

### Vault token lifetime
- Token already minted (1-year periodic, `orphan`, `cloudmanager` policy). Package installs
  `cloud-manager-token-renew.{service,timer}` running `vault token renew-self` weekly so it
  renews indefinitely.

### Bootstrapping — Layer 0 vs Layer 1 (the load-bearing contract)
- **Layer 0 (prerequisites, must exist first):** Postgres + RabbitMQ (satisfied via Debian
  `Depends:`, so apt installs/starts them before postinst); an **initialized + unsealed Vault**
  — a **deliberate prerequisite, never auto-`vault operator init`** (it produces unseal/root
  material a human must safeguard).
- **Layer 1 (this package) postinst — idempotent detect-and-adopt:**
  - Vault reachable+initialized+unsealed → adopt; create the `cloudmanager` policy / token
    only if missing. **Never init.**
  - Postgres `clouddb` exists → adopt, **never drop**. Seed/rotate the role password using the
    existing pg/rabbitmq playbook pattern (**read Vault → reuse if present, else generate
    32-char random**, set on the service, write to `cloudmanager/controller/postgres`).
  - RabbitMQ adopt vhost/user; rotate password the same way → `cloudmanager/controller/rabbitmq`;
    push the new value to **all three consumers** (API cred, vorch, porch).
  - Token `.cred`: encrypt only if absent; never overwrite.
  - `prerm`/`postrm`: stop/disable units only — never touch Vault data or `clouddb`.
- Fresh-install bootstraps Layer 1 atop a prepared Layer 0; adopt path runs safely on the
  already-provisioned controller (this box, after a Vault snapshot + `pg_dump` backup).

### Build tooling
- `debian/` per package (`control`, `rules`, `postinst`, `prerm`, `postrm`, `*.install`),
  `dpkg-buildpackage`/`debhelper`, `lintian`-clean. Binaries assembled from their source repos.
- Execution runs via `headless-run.py exec` against `ubuntu-server`, so every step must be
  **self-contained and idempotent** (safe to re-run after a teardown/recreate bootstrap).

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000 | Setup | `hv_status` + `hv_init`/`hv_next` → execution branch |
| 0001 | Packaging skeleton + build tooling | Pick the home repo; `debian/` for all three packages + `rules`/build assembling binaries. Installable `.deb`s, no secret logic yet. **Test:** `dpkg-buildpackage` + `lintian` clean; `apt install` lays files. |
| 0002 | systemd units + encrypted-credential wiring | Units with `LoadCredentialEncrypted` + `VAULT_TOKEN_FILE`; the .NET `AddKeyPerFile` layered-config change. **Test:** secret materialises at `$CREDENTIALS_DIRECTORY` (0400), nothing in `systemctl cat`, services read it. |
| 0003 | Idempotent postinst (Layer 0/1 detect-and-adopt) | Detect/adopt Vault+PG+RabbitMQ; encrypt token `.cred`; create policy/token + renew timer if missing; never re-init/drop. **Test:** run twice (second run no-op); fresh container with prepared Layer 0 bootstraps clean. |
| 0004 | Secret migration + password rotation | Rotate PG (API) and RabbitMQ (API+vorch+porch) passwords via the reuse-or-generate-to-Vault pattern; delete plaintext `Environment=` secrets; install the token-renew timer. **Test:** all services reconnect with rotated creds from creds/Vault; `systemctl cat` clean. |
| 0005 | Verification + cleanup sweep | **Adopt** path on the live controller after Vault snapshot + `pg_dump`; **fresh-install** in a throwaway 24.04 VM/container. Both end with all units active + an end-to-end VM provision (SSH key in Vault, IP assigned). Grep host + repo for stray plaintext secrets and clean up. |

## Open questions

1. **Packaging home repo:** new `cloud-manager-packaging` repo, or land `debian/` under
   the existing **`vm-infra/secret-management`** repo (already in the deck)?
2. **Web serving:** does nginx terminate `/api` locally, or stay behind the existing HAProxy
   on `:443`? (Affects the nginx site shipped in `cloud-manager-web`.)
3. **Fresh-install Layer 0:** is the Vault init/unseal bootstrap in scope for this package
   (as a separate `cloud-manager-vault-bootstrap` helper), or documented as a manual
   prerequisite for now?
4. **Renew cadence:** weekly `renew-self` against the 1-year period — confirm interval.
