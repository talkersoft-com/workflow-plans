# hv mcp run <name>

Idea: Add a `hv mcp run <name>` command to hive-deck that launches a registered MCP server by name directly from the hive CLI, without needing to know the full node path or profile flags.

## Vision
- `hv mcp run db` — starts database-toolkit MCP
- `hv mcp run cloud` — starts cloud-manager MCP
- `hv mcp list` — lists all registered MCPs with their namespaces and profiles

## Why it matters
Right now MCP servers are launched by specifying the full `node /path/to/dist/index.js --profile <name>` command in Cursor/Claude config. hive-deck knows the workspace layout, so it can resolve these paths automatically and expose a unified launcher. This would also allow hot-reloading configs and centralizing MCP registration in deck files.

## Rough design
- Add `mcps:` key to deck YAML (or setup config.yaml)
- Each entry maps a short name → namespace, profile, and build path
- `hv mcp run <name>` resolves the path, selects profile, spawns the server
