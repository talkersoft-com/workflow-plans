# Deck: cloud-manager

## Deck name
`cloud-manager`

## Repos with code changes
| Repo | What changes |
|------|-------------|
| `vorch-lib` | `provision/provision.go` `injectSecrets` + the generated `seal-credentials.sh`: version-aware seal (systemd-creds on ≥250, root-only `0600` credstore file on <250), guaranteed shred of `/run/cm-seed`, loud failure on seal error. |
| `cloud-manager-api` | `tests/hello-db-web/` consuming side: `deploy.sh` + `hello-db-web.service` pick `LoadCredential` vs `LoadCredentialEncrypted` by the VM's systemd version; `server.js` stays credential-only. **Do not touch `tests/test_secret_injection.py` assertions.** |

Repos not listed will be on the feature branch but skipped by hv_integrate.

## Branch
`serpentish-shuttlecock`

## Required deploy/verify steps (defined — not optional, not open)
- **Rebuild + redeploy `vorch-service` on the host** (`ubuntu-server`) so the new seal logic is live
  — the standard deploy script targets api/mcp/web only; rebuild + restart `vorch_service` on the host
  and record the exact command in `Results/RESULT.md`.
- **Run the acceptance test on ubuntu-server:**
  `cloud-manager-api/tests/test_secret_injection.py` — both cases must PASS, exit 0.

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
              message: "fix: secret-binding injection works on Ubuntu 22.04 and 24.04 (version-aware seal)"
              title:   "fix: secret-binding injection works on Ubuntu 22.04 and 24.04 (version-aware seal)"
              stage:   "exec"
```
