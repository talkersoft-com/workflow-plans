# Deck: cloud-manager

## Deck name
`cloud-manager`

## Repos with code changes
| Repo | What changes |
|------|-------------|
| `vm-infra/secret-management` *(candidate packaging home)* | New `debian/` for `cloud-manager-controller`, `cloud-manager-web`, `cloud-manager` metapackage: `control`, `rules`, `postinst`, `prerm`, `postrm`, `*.install`; the idempotent detect-and-adopt postinst; systemd units with `LoadCredentialEncrypted`; the `vault token renew-self` timer. |
| `cloud-manager-api` | Add the `AddKeyPerFile($CREDENTIALS_DIRECTORY)` layered config provider (env → cred → Vault) so `VAULT_TOKEN` / `DBPASSWD` / `RABBITMQ_PASSWORD` resolve from systemd credentials. |
| `cloud-manager-web` | nginx serving config (replace `serve`); reverse-proxy `/api` → `:5250`. |

Repos not listed will be on the feature branch but skipped by hv_ship.
`vorch-service`/`vorch-lib` need no code change (Go vault client already honours `VAULT_TOKEN_FILE`); they only get new unit/config wiring shipped via the packaging repo.

## Branch
`luscious-heartwood`

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
hv_ship  deck: "cloud-manager"
         message: "feat: Debian packaging + systemd-creds secret injection for the controller"
         title:   "feat: Debian packaging + systemd-creds secret injection for the controller"
         stage:   "exec"
```
