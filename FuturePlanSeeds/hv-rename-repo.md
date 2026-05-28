# hv rename-repo

Idea: Add a `hv rename-repo <org/old-name> <new-name>` command to hive-deck that renames a GitHub repository via the gh CLI and updates all references in local deck/config files.

## Vision
- `hv rename-repo talkersoft-com/database-toolkit talkersoft-com/hv-db-toolkit`
- Calls `gh repo rename` (or GitHub API) to do the actual rename
- Scans all `.hv/*.yaml` deck files for references and updates them
- Optionally updates remote URLs in locally cloned git repos

## Why it matters
Repos get renamed as projects evolve (e.g. the database-toolkit → hv-db-toolkit pattern). Doing this manually requires touching GitHub UI + grep/replace across all deck files + re-syncing git remotes. A single hive command can atomize this.

## Rough design
- `gh repo rename` for the GitHub side
- Walk all deck YAML files in `~/.hv/`, find `org/old-name` references, rewrite to `org/new-name`
- Walk `decks_root` subdirectories, find git repos matching old remote URL, run `git remote set-url origin <new-url>`
