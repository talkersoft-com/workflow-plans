# Ansible Install + Vault Integration Plan

Companion to `PLAYBOOK-INTEGRATION-PLAN.md`. That plan needs `ansible-runner`,
the `community.hashi_vault` collection, and a Vault policy in place before any
code lands. This plan covers getting those onto the server idempotently.

## Problem

Today the server has nothing Ansible-related installed. The playbook
integration plan assumes a worker can call `ansible_runner.run(...)`, that
playbooks can invoke `community.hashi_vault.vault_write`, and that Vault
has a policy permitting the orchestrator to mint short-lived child tokens
scoped to per-(vm, playbook) Vault prefixes. None of that exists yet.

## Goal

A pair of idempotent scripts — system install + Vault config — that can
run on a fresh server, leave it ready to execute playbooks, and are safe
to re-run after server rebuilds.

Match the existing tooling pattern (`cloud-manager-api/scripts/vault/cli/`
uses Python click + a YAML config). Add a new `cloud-manager-api/scripts/ansible/`
that follows the same shape.

## Layout

```
cloud-manager-api/scripts/ansible/
├── ansible-config.yaml          ← versions, paths, Vault settings (mirrors vault-config.yaml)
├── ansible-cli                  ← entry-point shell wrapper
├── cli/                         ← Python click commands
│   ├── main.py
│   ├── install.py
│   ├── uninstall.py
│   ├── configure.py             ← creates/refreshes Vault policy + role
│   └── verify.py                ← smoke test
├── scripts/                     ← per-operation impls
│   ├── install-ansible.py
│   ├── install-ansible.sh
│   └── _run.sh
└── policies/
    └── cloudmanager-playbook-runner.hcl
```

## Software to Install

Idempotent rules: every step checks first, installs only if missing.

| Component | Install method | Idempotency check |
|---|---|---|
| `python3-pip`, `python3-venv` | `apt-get install -y` | `dpkg -l` |
| `ansible-core` (>= 2.16) | `pipx install --include-deps ansible-core` | `command -v ansible-playbook && ansible --version` |
| `ansible-runner` | `pipx install ansible-runner` | `command -v ansible-runner` |
| `hvac` (Python Vault client) | `pipx inject ansible-core hvac` (so it lives in the same venv as the collection consumers) | `pipx runpip ansible-core show hvac` |
| `community.hashi_vault` Ansible collection | `ansible-galaxy collection install community.hashi_vault` | `ansible-galaxy collection list community.hashi_vault \| grep -q community.hashi_vault` |
| `git` | `apt-get install -y` | `command -v git` |

Use `pipx` (not system `pip`) so we don't pollute the system Python and
can pin Ansible's Python deps separately. `pipx install` is idempotent
when re-run.

## ansible-config.yaml

```yaml
ansible:
  version: "2.16.0"
  runner_version: "2.4.0"
  hashi_vault_collection_version: "6.2.0"
  hvac_version: "2.3.0"
  install_user: cloudmanager
  install_method: pipx                 # pipx | venv | system
  collection_path: /home/cloudmanager/.ansible/collections

vault:
  address: https://ubuntu-server.talkersoft.com:8200
  policy_name: cloudmanager-playbook-runner
  policy_file: policies/cloudmanager-playbook-runner.hcl
  # Token the install script uses to write the policy.
  # In practice: pass via env VAULT_TOKEN; never store in the YAML.

paths:
  playbook_workdir: /var/lib/cloud-manager/playbook-runs    # tmpfs preferred
  log_dir: /var/log/cloud-manager/playbook-runs
```

Mirrors `scripts/vault/vault-config.yaml`. Token deliberately not in YAML.

## Vault Policy

The orchestrator's existing `cloudmanager` token does not let it mint
arbitrary child tokens. We need a policy that does, plus a role to attach
that policy to runtime tokens.

### `policies/cloudmanager-playbook-runner.hcl`

```hcl
# Allow the orchestrator to mint scoped child tokens for playbook runs.
path "auth/token/create" {
  capabilities = ["update"]
}

# Bound by the role below — see "Token Role" — the child tokens themselves
# can only write under cloudmanager/vm/instances/<anything>/<anything>/*
# which is enforced via the role's allowed_policies template.

# Orchestrator also needs to list + read the secrets a run wrote
# (to surface them in the UI) and revoke its own child tokens.
path "cloudmanager/metadata/vm/instances/+/+/*" {
  capabilities = ["list", "read"]
}
path "cloudmanager/data/vm/instances/+/+/*" {
  capabilities = ["read"]
}
path "auth/token/revoke" {
  capabilities = ["update"]
}
```

### Token Role: `playbook-run`

```bash
vault write auth/token/roles/playbook-run \
  allowed_policies=cloudmanager-playbook-write-scoped \
  orphan=true \
  renewable=false \
  explicit_max_ttl=2h \
  token_type=service
```

Marks tokens minted under this role as orphan (so they survive parent
revocation — important if orchestrator restarts mid-run) and caps them
at 2h hard max.

### Per-run scoped policy: `cloudmanager-playbook-write-scoped`

Created **at install time, parameterized at run time** via Vault's
templated policies. The orchestrator passes the (vm_pub, pb_pub) pair as
metadata; the policy template substitutes them.

```hcl
# Templated policy — uses {{identity.entity.metadata.vm_public_id}} etc.
# Bind metadata when minting the child token.
path "cloudmanager/data/vm/instances/{{identity.entity.metadata.vm_public_id}}/{{identity.entity.metadata.playbook_public_id}}/*" {
  capabilities = ["create", "update", "read", "list"]
}
path "cloudmanager/metadata/vm/instances/{{identity.entity.metadata.vm_public_id}}/{{identity.entity.metadata.playbook_public_id}}/*" {
  capabilities = ["list", "read"]
}
```

This is the elegant version. If Vault's templated-policy syntax turns out
to be a pain in practice, the fallback is simpler:

**Fallback**: a single broad policy `cloudmanager-playbook-write` that
allows write across the whole `cloudmanager/data/vm/instances/+/+/*`
namespace. Child tokens get this policy and a short TTL. Less granular
(a buggy playbook could write outside its prefix) but no policy
templating. Recommended starting point; tighten later.

## Verify (Smoke Test)

`ansible-cli verify` runs:

1. `ansible --version` returns ≥ target version.
2. `ansible-runner --version` succeeds.
3. `ansible-galaxy collection list community.hashi_vault` shows expected version.
4. `python3 -c "import hvac; hvac.Client('https://ubuntu-server.talkersoft.com:8200').sys.read_health_status()"` succeeds.
5. Runs an in-memory no-op playbook against `localhost` via
   `ansible_runner.run(...)`. Exit code 0.
6. Mints a test child token from `playbook-run` role with throwaway
   metadata, writes a test key, reads it back, deletes it, revokes the
   token.

Each step prints PASS / FAIL with a one-line diagnosis.

## Health Probe (runtime)

The verify command is one-shot. For runtime health — used to drive
`feature_flags.playbooks.healthy` in `PLAYBOOK-INTEGRATION-PLAN.md` — the
worker runs a lighter probe:

- Every N minutes (default: 5), invoke `ansible_runner.run({}, playbook=<no-op>)`
  against `localhost` with `gather_facts: no`.
- On success: `UPDATE feature_flags SET healthy=true, last_probed_at=NOW() WHERE key='playbooks'`.
- On failure: `healthy=false`. Log error.
- Also re-probed at worker startup, so a worker restart re-evaluates
  immediately rather than waiting for the next tick.

This lets the orchestrator UI distinguish three states cleanly:

| `enabled` | `healthy` | Meaning                                                   |
|-----------|-----------|-----------------------------------------------------------|
| false     | (any)     | Feature is off — UI hides it                              |
| true      | true      | Working                                                   |
| true      | false     | Operator turned it on but the install is broken / missing |

## Opt-in Flow

The expected sequence on a fresh server:

1. App is installed and running. `feature_flags.playbooks.enabled=false`
   (seed value). Web UI shows no playbook anything.
2. Operator runs `ansible-cli install` → all software present.
3. Operator runs `ansible-cli configure` → Vault policy + token role present.
4. Operator runs `ansible-cli verify` → all PASS.
5. Operator turns the feature on:
   `curl -X PATCH /api/v1/admin/feature-flags/playbooks -d '{"enabled": true}'`
6. Worker's next health probe sets `healthy=true`; UI lights up.

If step 5 happens before step 2–4, the API returns 503 on playbook routes
and the UI shows an install-instructions banner — never silent breakage.

## Idempotency Guards

- `install.py` is safe to run any number of times. Each step is a check +
  conditional action.
- `configure.py` (Vault policy + role) uses `vault policy write` (which
  is upsert) and `vault write auth/token/roles/playbook-run` (also
  upsert).
- `uninstall.py` reverses everything but is **not** safe to run while
  workers are mid-run. Document the order: stop workers → uninstall →
  delete policy.

## Phases

1. **Install script** — pipx + collections, idempotent guards, verify
   command. No Vault touching yet.
2. **Vault policy provisioning** — `configure.py` writes the static policy
   + token role.
3. **Templated policy** — if Vault metadata-templated policies pan out in
   testing, swap in the scoped version. Otherwise stay on the broad
   policy.
4. **Health probe in worker** — periodic `ansible_runner.run` no-op that
   updates `feature_flags.playbooks.healthy`. Requires the feature-flag
   scaffolding from Phase 0 of `PLAYBOOK-INTEGRATION-PLAN.md` to exist.
5. **Smoke test in CI / on fresh box** — full reinstall path works from a
   blank Ubuntu; verify ends with flipping the flag on via the API.

## Open Questions

- **Install user**: the install scripts run as `cloudmanager` (matches
  the existing convention from `/kvm-automator/`). Worker should run as
  the same user. Confirm.
- **Where the worker process actually runs**: same box as the API today,
  separate worker pool eventually. Both should be fine with this install.
- **Collection path**: `~/.ansible/collections` per-user vs. a shared
  `/usr/share/ansible/collections`. Per-user keeps perms simple; shared
  helps if you ever run workers as multiple users.
- **Galaxy mirror / offline install**: scripts default to public Galaxy.
  If the org wants offline-only builds, swap in a Galaxy proxy (Nexus,
  Artifactory) — out of scope here.

## Related Plans

- **`PLAYBOOK-INTEGRATION-PLAN.md`** — the feature this enables.
- **`VAULT-SSH-KEY-PLAN.md`** (in `vorch-service/`) — original Vault SSH
  key flow that established the `cloudmanager/vm/instances/<vm_pub>/`
  prefix this plan extends.
