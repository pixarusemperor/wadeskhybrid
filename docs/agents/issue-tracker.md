# Issue tracker: GitHub

Issues and PRDs for this repo live as GitHub issues on `pixarusemperor/wadeskhybrid`. Use the `gh` CLI for all operations (repo inferred from `git remote -v`).

## Conventions

- **Create an issue**: `gh issue create --title "..." --body "..."`. Use a heredoc for multi-line bodies.
- **Read an issue**: `gh issue view <number> --comments`
- **List issues**: `gh issue list --state open --json number,title,body,labels,comments`
- **Comment on an issue**: `gh issue comment <number> --body "..."`
- **Apply / remove labels**: `gh issue edit <number> --add-label "..."` / `--remove-label "..."`
- **Close**: `gh issue close <number> --comment "..."`

## Wayfinding operations

Used by the `wayfinder` skill.

- **Map**: a single open issue labelled `wayfinder:map` — the canonical artifact.
- **Tickets**: child issues of the map, each carrying a `wayfinder:<type>` label (`research`, `prototype`, `grilling`, `task`).
- **Claim**: assign the issue to the dev resolving it, *before* any work. An open issue with no assignee is unclaimed.
- **Blocking**: GitHub Issues exposes no native dependency via `gh` — use the body convention: each ticket lists `Blocked by: #N` and `Blocks: #N`. A ticket is unblocked when every ticket in `Blocked by` is closed.
- **Frontier** (open, unblocked, unclaimed, excluding the map):

```bash
gh issue list --state open --json number,title,labels,assignees \
  --jq '.[] | select(any(.labels[].name; startswith("wayfinder:"))) | select(.assignees | length == 0) | select(any(.labels[].name; . != "wayfinder:map"))'
```

- **Resolution**: post the answer as a resolution comment, close the issue, and append a line to the map's "Decisions so far".
