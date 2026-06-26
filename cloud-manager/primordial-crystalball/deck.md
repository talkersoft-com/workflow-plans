# Deck: cloud-manager

## Deck name
`cloud-manager`

## Repos with code changes
| Repo | What changes |
|------|-------------|
| `cloud-manager-api` | `ReadKvSecretAsync` on IVaultClient/VaultClient (KV v2, cloudmanager mount); binding resolution at instantiate (MarketplaceController/ProvisionService/BlueprintRunSequencer) → read Vault → build resolved secrets; `Secrets` payload on `VmCommandData`; populate `vm.vm_secret_bindings` on provision success. Secret values never logged/returned. |
| `vorch-lib` | `Secrets` field on `createvm.VM`/`CommandData` (YAML lockstep with C#); extend `InitializeHost`/user-data builder to write base64 secret to tmpfs `write_files` + first-boot `runcmd` `systemd-creds encrypt --with-key=host` → `/etc/credstore.encrypted/<credName>` + shred plaintext; `credName` charset validation. |
| `vorch-service` | Thread the new secrets field through the create-vm consumer into `InitializeHost`. |

Repos not listed will be on the feature branch but skipped by hv_integrate.

## Branch
TBD — recorded at runtime in Task 0000.

## Dependency
**Requires workflow #1.5 (Secret Manager — `wheezy-cottonmouth`) merged first** (the `Secret`
entity, `secret_id` on bindings, and the empty `vm.vm_secret_bindings` table). Do not execute
this workflow until #1.5 is on the integration branch.

## Manual steps
**vorch redeploy:** the cloud-manager deploy script targets api/mcp/web only. `vorch-service` runs
on the hypervisor host and must be rebuilt + restarted there after merge (document the exact
command in Results/RESULT.md during exec).

## Initialize (Task 0000)
```
hv_status  deck: "cloud-manager"
hv_init    deck: "cloud-manager"
```
If already provisioned on a prior feature branch with all PRs merged:
```
hv_status  deck: "cloud-manager"
hv_next    deck: "cloud-manager"
```

## Ship
```
hv_integrate  deck: "cloud-manager"
              message: "feat: Secret Binding provisioning + injection (systemd creds via cloud-init)"
              title:   "feat: Secret Binding provisioning + injection (systemd creds via cloud-init)"
              stage:   "exec"
```
