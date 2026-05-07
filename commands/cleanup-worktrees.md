---
description: Remove leftover Claude Code worktrees under .claude/worktrees/.
allowed-tools: Bash(git worktree:*), Bash(gh pr view:*), Bash(gh pr list:*), Bash(git push:*), Bash(git branch:*)
---

Remove all leftover Claude worktrees in the current repo. The intent of this command is full cleanup — prompt for every worktree regardless of PR state.

Procedure:

1. Run `git worktree list --porcelain` to enumerate worktrees. Filter to ones whose path contains `/.claude/worktrees/`.
2. If none found, report "No Claude worktrees found" and stop.
3. For each, extract the branch (typically `claude/<name>`) and query its PR state:
   ```bash
   gh pr list --head <branch> --state all --json number,state,mergedAt,url --limit 1
   ```
4. Show the user a summary table: worktree path, branch, PR status (MERGED / CLOSED / OPEN / NO PR), PR URL. Include all worktrees.
5. Ask for confirmation: "Remove all of the above worktrees and their branches?" Single confirmation covers all of them.
6. On confirmation, for each worktree run:
   ```bash
   git worktree remove <path>
   git push origin --delete <branch>
   git branch -D <branch>
   ```
   If `git worktree remove` fails due to uncommitted changes, stop and report — do not use `--force`.
   If `git push origin --delete` fails (e.g. branch was never pushed), continue — don't abort.
7. Report what was removed and what remains.

Do not touch worktrees outside `.claude/worktrees/`. Do not delete branches that have no associated worktree in that directory.
