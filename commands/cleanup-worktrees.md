---
description: Remove leftover Claude Code worktrees under .claude/worktrees/.
allowed-tools: Bash(git worktree:*), Bash(gh pr list:*), Bash(git push:*), Bash(git branch:*), Bash(git fetch:*)
argument-hint: "[worktree-name]"
---

Remove leftover Claude worktrees in the current repo. With no argument, target every worktree under `.claude/worktrees/`. With an argument, target only the worktree whose path ends in `/$ARGUMENTS`.

Argument: `$ARGUMENTS`

If the current working directory is inside a `.claude/worktrees/` worktree, `cd` to the main repo root first (`git rev-parse --path-format=absolute --show-toplevel` from a non-worktree, or just walk up out of `.claude/worktrees/<name>/`) — `git worktree remove` refuses to remove the cwd.

Procedure:

1. Run `git fetch --prune origin` so PR-state and remote-tracking info is fresh and stale remote refs are cleared.
2. Run `git worktree list --porcelain` to enumerate worktrees. Filter to ones whose path contains `/.claude/worktrees/`. If `$ARGUMENTS` is non-empty, further filter to the single worktree whose path ends in `/$ARGUMENTS`.
3. If none found, report "No matching Claude worktrees found" and stop.
4. For each, extract the branch (typically `claude/<name>`) and query its PR state:
   ```bash
   gh pr list --head <branch> --state all --json number,state,mergedAt,url --limit 1
   ```
5. Show the user a summary table: worktree path, branch, PR status (MERGED / CLOSED / OPEN / NO PR), PR URL.
6. Ask for confirmation: "Remove the above worktree(s) and their branches?" Single confirmation covers all listed.
7. On confirmation, for each worktree:
   ```bash
   git worktree remove <path>
   git push origin --delete <branch>   # continue on failure (e.g. never pushed)
   git branch -d <branch>              # safe delete first
   ```
   - If `git worktree remove` fails due to uncommitted changes, skip that one and report — do not use `--force`.
   - If `git branch -d` fails because the branch is not merged: if PR state is `MERGED`, fall back to `git branch -D`. Otherwise stop on that branch and ask the user explicitly before force-deleting — unmerged work could be lost.
8. Run `git worktree prune` to clear any stale admin entries.
9. Report what was removed, what was skipped, and what remains.

Do not touch worktrees outside `.claude/worktrees/`. Do not delete branches that have no associated worktree in that directory.
