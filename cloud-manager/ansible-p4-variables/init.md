# Variable registry — Epic plan 4 (ansible-p4-variables)

## What this workflow does
Introduces the typed variable registry (C-VAR): `VariableDefinition` records with
required/default/internal kinds and plain/secret sensitivity, `SecretRef` Vault pointers in
place of values, the `$secretRef` override form enforced on every write surface, registry
harvest from P2's decomposition, and required-var gap APIs. Secrets ride the existing
SecretInputs → porch mechanism — no vorch/porch changes.

## Read before starting
- `deck.md`, `PLAN.md` — especially the `$secretRef` flow and the materialization expansion
- `../../master-plans/ANSIBLE-EPIC-CONTRACTS.md` — §7 C-VAR is binding verbatim; §11 invariants
- `../ansible-p1-inventory/PLAN.md`, `../ansible-p2-decomposition/PLAN.md` — the records this
  plan types and re-keys
- `../../master-plans/ANSIBLE-EPIC.md` — epic §4, §11

## Constraints
- **Series order**: requires P1 and P2 merged.
- **C-VAR is binding**: P5 and P8 consume the §7 shape verbatim — field names and the
  secret⇒null-default invariant must not drift.
- **No plaintext secrets**: DB CHECK + service validation; cleartext writes to secret-classified
  names are rejected, never silently accepted.
- No vorch-lib/vorch-service/porch changes; no web changes.
- `GET /run/{runId}/secret` stays excluded from MCP (standing intentional exclusion).
- Existing runs with untyped vars must behave byte-identically — regression-check including the
  blueprint sequencer path.
