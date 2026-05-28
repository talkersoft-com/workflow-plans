# Plan: hv_workflow — plan_path to plan_paths array

## Objective

Replace the single `plan_path` string parameter on the `hv_workflow` MCP tool with `plan_paths`, an array of absolute file paths. Every plan in the array is read and prepended to the workflow output in order, each separated by a `---` divider, before the workflow instructions begin. This lets a workflow run with full context from multiple related plans — for example a data-model plan and an API plan both prepended before a workflow scaffold.

## Background

`hv_workflow` currently accepts one `plan_path` (string). A single plan is read, prepended with a `## Plan` header and `---` separator, then the workflow output follows. This works for simple cases but breaks down when a workflow spans concerns documented in multiple plans (e.g. `DATA-MODEL-DELTAS.md` + `API-PLAN.md` both relevant to an execution workflow). The fix is purely in the MCP TypeScript layer — the Go CLI and fragment system are unchanged.

Current output shape (single plan):
```
## Plan
> Source: /path/to/PLAN.md
<plan content>

---

## Workflow: create-workflow
<workflow instructions>
```

Target output shape (multiple plans):
```
## Plan 1 of 2
> Source: /path/to/PLAN-A.md
<plan A content>

---

## Plan 2 of 2
> Source: /path/to/PLAN-B.md
<plan B content>

---

## Workflow: create-workflow
<workflow instructions>
```

## Design

### Parameter change

`workflow.ts` — replace:
```typescript
plan_path: z.string().describe(
  "Absolute path to the PLAN.md file..."
),
```

With:
```typescript
plan_paths: z.array(z.string()).describe(
  "Array of absolute paths to PLAN.md files to prepend as context before the workflow instructions. " +
  "Must be absolute paths (e.g. [\"/Users/you/workspace/cloud-manager/planning/workflow-plans/cloud-manager/thorny-catapult/PLAN.md\"]). " +
  "Plans are prepended in array order, each separated by --- so the agent reads all context before executing tasks. " +
  "At least one path is required."
),
```

### Output assembly

```typescript
// Read all plans, fail fast on first missing file
const planSections: string[] = [];
for (const planPath of plan_paths) {
  let content: string;
  try {
    content = readFileSync(planPath, "utf8");
  } catch (e) {
    return err(`Could not read plan file at ${planPath}: ${(e as Error).message}`);
  }
  const label = plan_paths.length > 1
    ? `## Plan ${planSections.length + 1} of ${plan_paths.length}`
    : `## Plan`;
  planSections.push(`${label}\n\n> Source: ${planPath}\n\n${content.trim()}`);
}

const combined =
  planSections.join("\n\n---\n\n") +
  `\n\n---\n\n## Workflow: ${name}\n\n` +
  output;
```

### Validation

Return `err` if `plan_paths` is empty (Zod `z.array().min(1)` handles this at schema level).

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000 | Setup | hv_status + hv_next |
| 0001 | Implement | Replace `plan_path` → `plan_paths` in `workflow.ts`; update assembly logic; `npm run build` |
| 0002 | Ship | Run tests, install, ship hive-deck-pro |
