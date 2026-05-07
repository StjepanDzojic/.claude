---
description: Remove Claude Code worktrees whose PRs have been merged or closed.
allowed-tools: Bash(git worktree:*), Bash(gh pr view:*), Bash(gh pr list:*)
---

Scan `.claude/worktrees/` in the current repo and remove any whose PR has been merged or closed.

Procedure:

1. Run `git worktree list --porcelain` to enumerate worktrees. Filter to ones whose path contains `/.claude/worktrees/`.
2. For each, extract the branch (typically `claude/<name>`) and query its PR state:
   ```bash
   gh pr list --head <branch> --state all --json number,state,mergedAt,url --limit 1
   ```
3. Classify each worktree:
   - **MERGED** — PR `state` is `MERGED`. Safe to remove.
   - **CLOSED** — PR `state` is `CLOSED` (not merged). Safe to remove, but flag separately.
   - **OPEN** — PR still open. Skip.
   - **NO PR** — branch has no PR. Skip and report; the user may want to handle manually.
4. Show the user a summary table: worktree path, branch, status, PR URL. List what would be removed.
5. Ask for confirmation before removing.
6. On confirmation, for each to-remove worktree run:
   ```bash
   git worktree remove <path>
   git branch -D <branch>
   ```
   If `git worktree remove` fails because of uncommitted changes, stop and report — do not use `--force`.
7. Report what was removed and what remains.

Do not touch worktrees outside `.claude/worktrees/`. Do not delete branches that have no associated worktree in that directory.
