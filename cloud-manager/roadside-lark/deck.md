# Deck: cloud-manager

## Deck name
`cloud-manager`

## Repos with code changes
| Repo | What changes |
|------|-------------|
| `vm-infra/secret-management` | The `cm-seed-secrets` script (reuse-or-generate → apply → Vault → `systemd-creds encrypt`); systemd unit drop-ins with `LoadCredentialEncrypted=` for api/vorch/porch; the live-cutover runbook. |
| `cloud-manager-api` | One change: `AddKeyPerFile($CREDENTIALS_DIRECTORY)` layered config provider (`env → cred`) so `VAULT_TOKEN` / `DBPASSWD` / `RABBITMQ_PASSWORD` resolve from decrypted cred files. |
| `vorch-service` | RabbitMQ cred wiring — read the rotated RabbitMQ password from a cred (or seeder-updated `config-server.yaml`). |

Repos not listed will be on the feature branch but skipped by hv ship.
`vorch-lib` needs no code change (Go vault client already honours `VAULT_TOKEN_FILE`).

## Branch
`roadside-lark`

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
              message: "feat: systemd-creds secret injection for the controller"
              title:   "feat: systemd-creds secret injection for the controller"
              stage:   "exec"
```
