# Plan: Flatten hive-deck MCP Profiles to decks + workflows

## Objective

Replace the three-level profile tower in hive-deck MCP's `profiles.json` with two flat profiles — `decks` (all deck-management tools) and `workflows` (extends decks, adds workflow tool) — matching the pattern used by cloud-manager-mcp. New tools are added to the correct profile with no new profile entry needed. Update `MCP_PROFILE` references from `hv.deck.pro` to `workflows` in all `.mcp.json` configs.

## Background

The current profiles.json has three levels of inheritance:
```
hv.deck (status, list, decks)
  └── hv.deck.operator (+ init, ship, sync, default, stash, unstash, list-pulls)
        └── hv.deck.pro (+ teardown, prune, mcp, workflow, await-merge)
```

Every new tool requires adding a profile entry or touching an existing one — there's no obvious "full" profile. Cloud-manager-mcp solves this with two profiles: a base that has all core tools, and a specialization that extends it. The hive-deck equivalent is `decks` (everything) and `workflows` (decks + workflow tools). The `workflows` profile is what every Claude Code session uses.

The `activeProfileName()` in `profiles.ts` defaults to `process.env.MCP_PROFILE ?? "hv.deck.pro"`. That default and all `.mcp.json` configs that set `MCP_PROFILE: "hv.deck.pro"` must be updated to `"workflows"`.

## Design

### New profiles.json

```json
{
  "profiles": {
    "decks": {
      "tools": [
        "status", "list", "decks",
        "init", "ship", "sync", "default",
        "stash", "unstash", "list-pulls",
        "teardown", "prune", "mcp", "await-merge"
      ]
    },
    "workflows": {
      "extends": "decks",
      "tools": ["workflow"]
    }
  }
}
```

Adding a new deck-management tool → add to `"decks"` tools array.  
Adding a new workflow tool → add to `"workflows"` tools array.  
No new profile entry ever needed.

### profiles.ts default update

```typescript
return process.env.MCP_PROFILE ?? "workflows";
```

### .mcp.json updates

Any `.mcp.json` that contains `"MCP_PROFILE": "hv.deck.pro"` must be updated to `"MCP_PROFILE": "workflows"`. Files to check:
- `~/workspace/cloud-manager/.mcp.json`
- `~/workspace/.mcp.json`
- `~/workspace/hive-deck-pro/.mcp.json`
- Any other workspace `.mcp.json`

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000 | Setup | hv_status verify |
| 0001 | profiles.json + profiles.ts | Replace tower with two profiles; update default profile name |
| 0002 | .mcp.json updates + rebuild + ship | Find and update all MCP_PROFILE references; npm run build; ship |
