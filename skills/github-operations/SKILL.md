---
name: github-operations
description: Use whenever you interact with GitHub — the `gh` CLI, pull requests, issues, releases, tags, branches, reviews, comments, labels, milestones, branch protection, or rulesets. Covers the PR lifecycle (rebase, raise, review, merge), issue and release conventions, URL formatting, GitHub Action version checks, and never-push-to-main. Fires on mentions of gh, GitHub, PR, pull request, issue, release, tag, review, merge, branch protection.
---

# GitHub Operations

Every interaction with GitHub — via `gh`, the API, or the web — goes through these rules.

---

## 1. Output and link format

- After any `gh` command that produces a URL (PR, issue, release, commit, workflow run), **output the URL as a clickable markdown link** in the chat response.
- **Always use full URLs** for PRs and issues: `https://github.com/owner/repo/pull/123`. Never shorthand like `owner/repo#123` — it doesn't render as a link.
- Example: `[mg-infrastructure#1234](https://github.com/prototypdigital/mg-infrastructure/pull/1234)`.

---

## 2. Branches

- **Never push directly to `main`.** Every change goes through a PR. In infrastructure repos the rule is absolute — `main` pushes auto-apply via TFC.
- Branch naming: `feat/…`, `fix/…`, `chore/…`. One task per branch.
- After a PR is opened, don't commit further unrelated changes to that branch — open a new branch.

---

## 3. Pull Requests

### Routing decisions — ask, don't pick silently

Before you start a commit/push flow, resolve where the change lands. When there's more than one plausible answer, render an `AskUserQuestion` per the `interactive` skill — do not decide silently. Typical cases:

- New work could extend an **open PR** vs go on a **new branch/PR** (related scope, same reviewer, CI cost).
- Fix could **amend** the last commit vs land as a **new commit** (the repo's squash/merge policy decides, but the user's intent wins when unclear).
- Change could pair with the **current feature branch** vs a **separate chore branch** (e.g. lint/config fixes discovered mid-feature).
- PR title/body may need updating to reflect added scope — confirm before rewriting.

Offer 2–3 options with the recommended one first and a one-line trade-off per option. "Other" is auto-injected by the UI; don't add it manually. Skip the prompt only when the user's most recent instruction already pinned the answer.

### Before raising

**Always rebase onto `origin/main`** (or the default branch) immediately before `gh pr create`, no exceptions, even if the branch was just created from `origin/main`:

```sh
git fetch origin
git rebase origin/main
git push --force-with-lease
```

Never raise a PR from a branch that hasn't been rebased onto the current `origin/main`.

### Creating the PR

- Title: short, under 70 chars.
- Body: use the repo's template if one exists (`.github/PULL_REQUEST_TEMPLATE.md`); otherwise a `## Summary` + `## Test plan` shape.
- **Never add the "🤖 Generated with Claude Code" footer** to PR descriptions.
- **Never escape backticks in heredoc PR bodies.** Always pass the body via `<<'EOF'` (single-quoted delimiter). Inside a single-quoted heredoc, no shell interpretation occurs — write bare backticks (`` ` ``), not `\``. Escaped backticks render literally in GitHub and break markdown formatting.
- If the repo has a plan-validator or similar pre-PR check, run it first.

### After raising

- **Before every push to a PR branch**, verify the PR is still open: `gh pr view <N> --json state --jq .state`. If it is `MERGED` or `CLOSED`, do not push — open a new branch off `main` instead.
- Output the PR URL as a clickable markdown link.
- If the repo requires labels (e.g., `bump:patch|minor|major` in `terraform-modules`), apply them immediately — don't wait for a reviewer to ask.
- If you make further changes after the PR is open, **update the PR description to match the current diff** — don't leave it describing reverted work.

### Reading review comments

- Line-level comments: `gh api repos/owner/repo/pulls/123/comments`
- Top-level review comments: `gh api repos/owner/repo/issues/123/comments`
- Reviews (approve/request-changes summaries): `gh api repos/owner/repo/pulls/123/reviews`

---

## 4. Issues

### Routing decisions — ask, don't pick silently

Before filing, resolve whether the report belongs as:

- A **new issue** vs a comment on an existing open issue (search first).
- An **issue** vs a **PR** (if the work is ready to ship, skip the issue).
- Filed **on this repo** vs an upstream/related repo.

When more than one answer is plausible, render an `AskUserQuestion` per the `interactive` skill. Skip the prompt only when the user's most recent instruction already pinned the answer.

### Before filing

- Search existing issues first: `gh issue list --search "<keywords>" --state all`. Link to a duplicate rather than re-filing.
- Use the repo's template if one exists (`.github/ISSUE_TEMPLATE/`). If multiple templates exist, pick one — don't file template-less when templates exist.

### Creating the issue

- Title: short, under 70 chars, no trailing punctuation. Start with a verb for action items ("Fix flaky …", "Add support for …"); use a noun phrase for bug reports ("Login redirect loop on Safari").
- Body: follow the template if present. Otherwise use `## Context` + `## Expected` + `## Actual` for bugs, or `## Problem` + `## Proposal` for features/chores.
- **Never add the "🤖 Generated with Claude Code" footer** to issue bodies.
- **Never escape backticks in heredoc issue bodies.** Same rule as PRs — single-quoted `<<'EOF'`, bare backticks.
- Apply labels at creation time if the repo uses them (`--label bug`, `--label feature`). Don't wait for triage.
- Set milestone, project, or assignee only when explicitly asked.

### Linking

- From a PR: "Closes #N", "Fixes #N", or a full URL (full URL preferred cross-repo).
- From another issue: use the full URL so it renders as a link in both directions.
- Output the new issue URL as a clickable markdown link in the chat response.

### Commenting, reading, closing

- Comment: `gh issue comment <N> --body "…"`.
- Read comments: `gh api repos/owner/repo/issues/<N>/comments`.
- Close with a reason: `gh issue close <N> --reason completed|not_planned`. Use `not_planned` for wontfix / duplicate / out-of-scope, and leave a comment explaining why before closing.
- When closing as a duplicate, link to the surviving issue in the closing comment.
- Don't auto-close issues opened by others without explicit instruction.

---

## 5. Releases and tags

### Discovery

- List recent releases: `gh release list --repo owner/repo --limit 5`. Use this before pinning any GitHub Action — never assume a major tag is current.
- Tag conventions are per-repo; inspect `git tag -l | head` before choosing a format.

### Creating a release

- Create: `gh release create vX.Y.Z --title "…" --notes "…"` (or `--generate-notes` to auto-compose from commits, or `--notes-file path`).
- Title: `vX.Y.Z` by default, matching the tag. Only deviate when the repo already uses a different convention (check the last few releases).

### Release body

- If the repo uses **release-please / semantic-release / auto-generated notes**, leave the generated body alone — don't hand-edit. Your job is to land conventional commits upstream and let the automation write the notes.
- When writing the body yourself:
  - Structure: `## What's changed` (bulleted, one line per change), then `## Breaking changes` (only if any), then `## Upgrade notes` (only if action required).
  - Each bullet links to the PR: `- Add retry on 429 ([#123](https://github.com/owner/repo/pull/123))`.
  - **Never add the "🤖 Generated with Claude Code" footer** to release bodies.
  - **Never escape backticks in heredoc release bodies.** Same rule as PRs and issues — single-quoted `<<'EOF'`, bare backticks. Release notes are the most code-fence-heavy of the three, so this matters most here.
- Output the release URL as a clickable markdown link in the chat response.

### Action repos

- `prototypdigital/action-*`: tag-based releases drive version consumption — align `action.yml` and the release workflow before tagging.
- Always move the major tag (`v1`, `v2`) to the new SHA after a compatible release: `git tag -f v1 vX.Y.Z && git push origin v1 --force`. Skip this on breaking releases — bump the major tag instead.

---

## 6. Reviews

- Request a review: `gh pr edit <N> --add-reviewer <user>`.
- Address review comments by pushing new commits (never amend/force-push after review has started unless explicitly requested).
- Re-request review after addressing: `gh pr edit <N> --add-reviewer <user>` again.

---

## 7. Branch protection and rulesets

- Managed via `gh api` or the repo settings UI.
- `prototypdigital/*` repos use the standard "Protect main" ruleset — applied by `scaffold-repo` / `scaffold-action-repo` at creation time. Don't modify without asking.

---

## 8. When in doubt

- Reading info about a GitHub resource by URL: use `gh` — don't scrape the web UI.
- If a `gh` command asks for interactive input, re-run with explicit flags; don't leave interactive prompts hanging.
- Destructive operations (delete branch/repo/release, force-push to shared branches, close+lock) require explicit user confirmation every time.
