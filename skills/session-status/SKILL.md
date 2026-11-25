---
name: session-status
description: Report what's actually left at the end of a session by re-verifying external state, not by recalling it. Use whenever the user asks "any leftovers?", "is this done?", "are we finished?", "what's left?", "session status", "wrap up", "where are we", or any equivalent end-of-session check. Never infer a PR, CI, or file state from earlier turns in the conversation — always re-run the relevant command.
---

# Session status checker

Purpose: at the end of a session, answer "what's still open?" accurately. The failure mode this prevents is **memory-guessing**: reporting that a PR is "waiting to merge" because you raised it two hours ago, when the user merged it ten minutes later outside the chat.

## Core rule

**Never infer external state from the transcript.** Every time this skill runs, re-check:

- PR state via `gh pr view <N> --repo <owner>/<repo> --json state,mergedAt,mergeable,statusCheckRollup`
- Local working tree via `git -C <repo> status --short` and `git -C <repo> branch --show-current`
- Remote branches via `gh` / `git ls-remote` if relevant
- CI via `gh pr checks <N> --repo <owner>/<repo>`
- External systems you touched (TFC runs, Notion pages, Slack messages): verify freshly via their API

Quoting `gh` output from earlier in the conversation as authoritative is forbidden. PR #1234 being OPEN three hours ago tells you nothing about its state now.

## Workflow

### 1. Scan the full session transcript

Read the whole conversation (not just the last few turns). Catalogue every item that could still be open:

- **PRs raised** — collect all `gh pr create` calls and the returned URLs.
- **Branches created** — any `git checkout -b` that produced work; was it pushed? Merged?
- **Questions the assistant asked the user that were never answered** — real open questions, not rhetorical checkpoints.
- **Tasks the assistant started but didn't finish** — e.g. "wiki submodule not added yet", "docs-sync port pending".
- **External actions** — API writes, scheduled jobs, MCP tool calls with side effects.
- **Untracked / uncommitted files** produced during the session.

### 2. Re-verify each item, item-by-item

For each PR:

```bash
gh pr view <N> --repo <owner>/<repo> --json number,state,mergedAt,mergeable,url,statusCheckRollup
```

Map `state` to one of: `MERGED`, `OPEN`, `CLOSED`.

For each local repo touched:

```bash
git -C <repo-path> status --short
git -C <repo-path> branch --show-current
```

For each CI-dependent PR still open:

```bash
gh pr checks <N> --repo <owner>/<repo>
```

Run these fresh — do not reuse output from earlier in the session.

### 3. Report

Use this output shape:

```markdown
## Session status

### Done
- <item> — <evidence, e.g. "MERGED 2026-04-18 20:47 UTC">

### Still open
- <item> — <actual state>, <recommended next step>

### Unanswered questions
- <verbatim question, or "none">

### Loose state
- <untracked files, uncommitted edits, abandoned branches, etc.>
```

If a category is empty, omit the heading.

### 4. If anything differs from what you said earlier in the session

Call it out explicitly. Example: "Earlier I reported PR #1214 as waiting to merge — it was actually merged at 20:47 UTC. Correcting now."

Do not paper over the drift. The user is relying on this skill precisely because earlier claims may be stale.

## Anti-patterns — forbidden

- "Probably merged by now" / "should be done"
- "Based on what we did earlier, [X] is still open" (without re-checking)
- Quoting earlier `gh pr view` output as current
- Reporting a PR as `OPEN` because you don't remember it being merged
- Skipping the `git status` re-check on the assumption that nothing changed
- Conflating "I raised this" with "this is still open"

## When to invoke

Trigger on any end-of-session wrap-up phrasing. Non-exhaustive:

- "any leftovers?"
- "is this done?"
- "are we finished?"
- "what's left?"
- "session status"
- "wrap up"
- "where are we"
- "anything still open?"
- "summary of today"

Also invoke proactively when the user signals they're about to stop ("I'm logging off", "end of day"), provided there was meaningful work in the session.
