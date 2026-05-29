# Plan: pr_mode-aware post-ship instructions in post-ship-reminder fragment

## Objective

Update `fragments/post-ship-reminder.md` so the instructions shown after a planning ship match the deck's `pr_mode`. When `pr_mode` is `await_merge` or `auto_merge`, hv_ship handles the merge and transition automatically — Claude should proceed immediately with execution. When `pr_mode` is `manual`, Claude shows the manual steps: merge the PR, run hv_next, then /loop.

## Background

`post-ship-reminder.md` currently shows the same "merge PR, run hv_next, then /loop" block regardless of `pr_mode`. This is wrong for `await_merge` and `auto_merge` — in those modes hv_ship is already polling and will auto-transition, so telling the user to manually merge and run hv_next is misleading. The `.PRMode` field is already in `TemplateData` and available in every fragment.

## Design

Replace the static content of `post-ship-reminder.md` with a Go template `{{if}}`/`{{else}}` block:

```
{{if eq .PRMode "manual"}}
**Planning shipped — [PR #<N>](<pr-url>)**

Next steps:
1. Merge the PR above
2. Then run `hv_next  deck: "{{.Deck}}"` to transition to the execution branch
3. Then start execution by running:

```
/loop Continue executing tasks in ...
```

Replace `<execution-branch>` with the branch the deck transitioned to after ship.
{{else}}
**Planning shipped — [PR #<N>](<pr-url>)**

hv_ship is handling the merge and branch transition automatically (pr_mode: {{.PRMode}}).
Once the PR merges and the deck transitions, continue immediately — no manual steps needed:

```
/loop Continue executing tasks in ...
```
{{end}}
```

The `/loop` command path is the same in both branches — only the surrounding instructions differ.

## Implementation phases

| Phase | Title    | Description                                                        |
|-------|----------|--------------------------------------------------------------------|
| 0000  | Setup    | hv_status + hv_next                                                |
| 0001  | Fragment | Update `fragments/post-ship-reminder.md` with `{{if}}` conditional; verify with `hv workflow hive-deck-pro create-workflow` under each pr_mode |

## Open questions

None.
