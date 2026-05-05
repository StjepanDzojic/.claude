---
name: amend-open-pr
description: Use whenever the plan for work inside an already-open PR changes but the PR's scope does not. Covers replacing commits, converting to draft, rewriting the PR body, dropping and re-adding files, changing rollout order. Fires on close-and-reopen, raise-a-new-PR-for-this, start-over, or any reflex to abandon an open PR whose purpose is still valid.
---

# Amend the Open PR, Don't Close It

When the **approach** to a task shifts but the **PR's stated scope** still holds, always amend the existing PR. Closing a PR is for scope changes, abandonment, or mistakes — not for in-flight course corrections.

---

## The rule

If the PR title still describes what the work is supposed to accomplish, the PR stays open. Everything else — commit contents, file list, sequencing, rollout notes, draft/ready state — is amendable.

**Scope unchanged** means: the one-line "why this PR exists" is still true.

- ✅ "Remove development environment from infra" → still true even if the commit flips from deleting the directory to adding `removed` blocks, or if the PR becomes a draft pending a destroy step.
- ❌ "Remove development environment from infra" → no longer true if we pivot to "Rename development to sandbox." That's a new PR.

---

## What to do instead of closing

| User signal | Amend action |
|---|---|
| "This needs to wait for X first" | `gh pr ready --undo` (convert to draft), update body with blocking step |
| "This approach won't work, do it differently" | Rewrite the commit(s) on the branch, force-push, refresh the PR body |
| "Split this out" | Keep the PR focused; move the split-off work to a *new* branch/PR, and remove it from this PR's diff |
| "Do it later / as a follow-up" | Draft the PR and note the prerequisite in the body — don't close |
| "Actually merge this first, then do Y" | No amendment needed — just sequence |

---

## Force-push is fine

On your own unmerged PR branch, force-pushing to rewrite history is the normal way to amend. Exceptions:
- Never force-push to `main` or a protected branch.
- If someone is actively reviewing, say so before force-pushing so they don't lose their place.

---

## When closing *is* correct

- The work is being abandoned outright.
- The PR's scope was wrong from the start (e.g. wrong repo, wrong target branch, duplicate of another PR).
- The branch will never be needed again.

In those cases, close with a comment explaining why, so the history is readable.

---

## Heuristic before any `gh pr close`

Ask: *"Does the PR title still describe work we intend to do?"*
- **Yes** → amend (body, commits, draft state). Don't close.
- **No** → close is appropriate.
