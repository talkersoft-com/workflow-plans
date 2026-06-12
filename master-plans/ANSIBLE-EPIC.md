# Master Requirements — Ansible MCP + Database-Backed Reusable UI

## 1. Purpose & Scope

Build a system that exposes Ansible operations through an MCP server, backed by a database that stores every reusable entity (roles, playbooks, tasks, variables, inventories), and fronted by a UI that maximizes reuse, makes variable/parameter resolution explicit, gives special treatment to sensitive parameters (tokens, secrets, vault refs), and renders/edits YAML chunks with modern, Ansible-aware syntax highlighting.

Core goals:
- Make roles and any other entity **reusable** and recomposable.
- Make variable resolution **transparent** — show the effective value and its provenance in the context of a specific operation.
- Treat **tokens and secrets** as first-class, specially-handled objects.
- Keep authoring **YAML-native** but structured underneath, with round-trip integrity.

---

## 2. Data Model & Storage

- **Decompose into addressable records.** Store roles, playbooks, plays, tasks, handlers, variables, inventories, host vars, and group vars as discrete, individually-referenceable records rather than flat YAML blobs.
- **Version everything.** Every entity carries history with diffing and rollback. Support "what changed between runs / between versions."
- **Explicit relationship graph.** Model edges directly: role → depends-on → role, playbook → includes → role, var → consumed-by → task, inventory → targets → host. This dependency graph powers reuse, impact analysis, and safe deletion.
- **Reverse indexes ("used-by").** For any entity, resolve every consumer before edit or delete. This prevents the stale-reference failure class (e.g. deleting a thing still referenced elsewhere, leaving dangling configs).
- **Round-trip fidelity.** Decompose → store → recompose must preserve comments, key order, and formatting. Use a round-trip-preserving parser (`ruamel.yaml`) server-side; never normalize away the user's formatting.

---

## 3. Reusability of Roles & Entities

- **Roles as first-class library objects** with metadata: description, tags, supported platforms, required vars, default vars, maintainer, version.
- **Clean parameter interface.** Separate each role's variables into *required inputs*, *defaults*, and *internal* vars so the role has a well-defined "interface" that can be satisfied from any playbook.
- **Composition & dependencies.** Support role composition/inheritance and dependency chains, with cycle detection.
- **Reuse beyond roles.** Task blocks, var sets, and inventory groups should also be reusable and referenceable, not copy-pasted.
- **Impact-aware editing.** Before editing/deleting a shared entity, surface "used by N playbooks/roles" with the list, and warn on breaking changes to a role's interface.

---

## 4. Variables & Parameters

- **Variable registry** with full precedence awareness. Ansible has a deep (~22-level) precedence order; the system must know it and be able to compute the winner.
- **Sensitive-parameter typing.** Tokens, API keys, passwords, and vault-encrypted values are explicitly typed and flagged as secrets, with distinct rendering and masking everywhere they appear.
- **No plaintext secrets.** Secrets are *referenced*, never stored in the DB in cleartext. Integrate with Ansible Vault and/or an external secret store; the DB holds references/pointers only.
- **Required-var enforcement.** Track which vars a role/operation requires; flag missing ones before run.

---

## 5. Contextual Value Resolution

- **Effective-value preview.** For any variable, resolve its effective value for a specific inventory + host + playbook combination *before* running.
- **Provenance.** Show where the winning value came from (group_vars, host_vars, role default, extra-vars, vault) and what it overrode.
- **Operation context.** "Read the value in the context of an operation" — resolution is always scoped to the run that's being assembled, not abstract.
- **Gap detection.** Flag undefined/unresolved vars and missing required parameters per target, ideally wired to `--check` / dry-run.

---

## 6. MCP Integration Layer

- **Stable tool surface.** Define MCP tools with stable schemas: list/get roles & playbooks, resolve vars (with provenance), lint, dry-run (`--check`), execute, fetch run status, stream output.
- **Async execution.** Playbook runs are long-running and not request/response. Use a **spawn-and-return `job_id`** pattern with status polling; do not block the MCP call for the duration of a run. (Backgrounding belongs server-side, not in the transport.)
- **Streaming output.** Stream/chunk stdout/stderr through the job rather than buffering an entire run into one response.
- **Idempotent, safe tools.** Read tools are side-effect-free; mutating/executing tools are clearly separated and auditable.

---

## 7. YAML Editing & Syntax Highlighting

- **Real editor component.** Use Monaco or CodeMirror 6 (not regex highlighting). CodeMirror 6 is lighter/more embeddable; Monaco gives a fuller VS Code-style experience (hover, completion) if desired later.
- **YAML-structure highlighting.** Keys vs. values, anchors/aliases (`&`/`*`), block scalars (`|`, `>`), comments, indentation guides.
- **Ansible semantics layered on top.** Highlight module names and `vars:`/`tasks:`/`handlers:` keywords distinctly.
- **Embedded Jinja2.** `{{ }}` expressions inside YAML strings are highlighted as template expressions via nested/embedded language support — not as plain strings.
- **Fragment tolerance.** Chunks edited out of context may be invalid standalone YAML; configure the parser to tolerate fragments or virtually wrap them so highlighting/linting doesn't error on a "missing document." Preserve relative indentation and round-trip back into the parent cleanly.
- **Inline lint/validation.** Run `ansible-lint` / `yamllint` and surface results as inline squiggles + gutter markers, plus schema-aware checks (unknown modules, missing required args, bad indentation).
- **Round-trip integrity at the editor.** Treat the editor buffer as source of truth; diff against it so DB decomposition never mangles formatting, comments, or key order.

---

## 8. Special Highlighting — Tokens, Secrets & Required Vars

- **Semantic decorations.** Beyond base syntax coloring, add editor decorations/semantic tokens driven by DB metadata: mark sensitive parameters, vault refs, and required vars inline with distinct color/badge/gutter icon. (Both Monaco and CodeMirror support custom decoration layers.)
- **Meaningful, not cosmetic.** `{{ api_token }}` should look different from `{{ hostname }}` based on its classification, not just generic string color.
- **Hover provenance overlay.** On hover over a var, show its resolved value + provenance inline (secrets redacted). This is the high-value differentiator: highlighting that *means something*.
- **Masking everywhere.** Secret values masked in editor, previews, logs, and run history.

---

## 9. UI / UX

- **Library browser.** Role/playbook browser with search, tags, and a visualized dependency graph.
- **Playbook composer.** "Build a playbook" flow: pick roles; the UI surfaces each role's required parameters as a form; validate before save.
- **Effective-config preview.** Per-target pane showing resolved vars + provenance before execution.
- **Secret indicators.** Lock icons, masked fields, "set via vault" indicators wherever sensitive parameters appear.
- **Run console.** Live streaming output view tied to the async `job_id`, with status, progress, and per-task results.

---

## 10. Execution, Safety & Observability

- **Idempotency visibility.** Surface which tasks are naturally idempotent vs. guarded, and show check-mode results.
- **Guardrails.** Confirmation on destructive ops; "used by N playbooks" warnings; lint/syntax-check gates before run.
- **Full audit trail.** Run history records who ran what, against which inventory, with which resolved vars — secrets redacted. Retain logs and outcomes per run.
- **Dry-run first.** Encourage/allow `--check` before real execution from the same UI flow.

---

## 11. Cross-Cutting Requirements

- **Security.** Secret references only; least-privilege access to vault/secret store; redaction in all surfaces; auditable mutating actions.
- **Performance.** Resolution and effective-value previews should be responsive even with large inventories and deep precedence chains.
- **Extensibility.** Stable MCP schemas and a clean entity model so new entity types, lint rules, or secret backends can be added without rework.
- **Data integrity.** Round-trip preservation and reverse-index consistency are invariants, not nice-to-haves.