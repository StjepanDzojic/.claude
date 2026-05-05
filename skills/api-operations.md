---
name: api-operations
description: Use whenever an action requires talking to an external API or CLI — Notion, GitHub, AWS, Slack, Linear, Terraform Cloud, etc. Covers tool selection (MCP > official CLI > API), credential storage, prompting before API-backed actions, and the two-factor rule for destructive operations.
---

# API Operations

> **Interactive asks.** When this skill tells you to ask the user anything with discrete options (packaging choice, two-factor confirmation with alternatives, etc.), render it via `AskUserQuestion` per the `interactive` skill — never as chat-based numbered options.

Use this skill any time you're about to perform an action that hits an external service — reading, writing, creating, updating, deleting, uploading, sending a message, triggering a run, merging a PR, etc. It applies equally whether you reach the service through an MCP server, a CLI, or a raw REST call.

---

## 1. Pick the right tool

Preference order for every external service:

1. **Official MCP server**, if one is configured and supports the action.
2. **Official CLI**, installed via Homebrew.
3. **API** (curl / SDK) with a token.

### MCP is preferred over CLI when all of the following hold

For any given action, pick the **MCP** over the CLI if the MCP:

- **Produces a cleaner result** — structured data you can consume directly, no string-parsing of CLI output, no re-shaping.
- **Is more performant** — fewer round-trips, no shell startup, no per-call auth handshake.
- **Uses fewer tokens while still getting the job done** — returns the specific fields you need instead of a fixed verbose CLI dump.

If any of those is false for the action at hand (e.g. the MCP returns a page of noise you'd have to trim down, or it can't do what the CLI can in one step), use the CLI instead. The decision is per-action, not per-service — some Notion ops fit MCP cleanly, others don't.

Within the same service, it's fine to mix: MCP for reads and CLI/API for a specific write the MCP doesn't expose cleanly. Announce the split if you do it.

### 1d. Suggesting a new MCP

If an MCP for the service does **not** currently exist on this machine, but you know of one that would make future operations meaningfully better, you may proactively suggest adding it. Do **not** install or configure anything without approval.

When you suggest, surface — in one message, clearly labelled:

- **Name and source** of the MCP (official vendor, community project, etc.), and how auth works.
- **Pros over the current CLI/API path** — expected token savings, performance wins, cleaner auth (OAuth vs static PAT), structured output, richer operations.
- **Cons / risks** — extra dependency, trust in a third-party server, possible feature gaps, lock-in, setup friction.
- **When the CLI will still be used** — e.g. destructive ops the MCP refuses, file uploads, edge-case flags.
- **Packaging choice** — ask the user whether the integration should live as:
  - a **skill**, if multiple agents will reuse it, or
  - a **subagent**, if it cleanly encapsulates a single responsibility,

  and let them decide. Never pick for them.

Suggestions are standalone turns. Do not chain a suggestion with an unrelated tool call.

### 1a. CLI discovery

Almost every CLI on this machine is installed via Homebrew. When you need a CLI:

1. Try running it directly from its Homebrew location first: `/opt/homebrew/bin/<tool>` or `/usr/local/bin/<tool>`.
2. If that fails, run `which <tool>` to see if it's elsewhere on `$PATH`.
3. If both fail, check whether the service has an official CLI. If it does and it's on Homebrew, prompt the user to install it — don't silently fall back to the API.
4. Never assume a tool is not installed without steps 1–3.

### 1b. CLI sessions vs. tokens

If the CLI has its own login/session management (e.g. `gh auth login`, `aws sso login`, `op signin`), **prefer that over a long-lived PAT**. Sessions are revocable, auditable, and usually shorter-lived.

Only fall back to a token-in-env approach when the CLI doesn't manage sessions, or there is no CLI at all.

### 1c. Token storage

When a service requires a long-lived token (e.g. `NOTION_TOKEN`, `LINEAR_API_KEY`):

- Store it in `~/.claude/env` as `KEY=value` (one per line, no `export`).
- Expect it to be loaded into the environment automatically by the user's shell. If an agent needs the value and the env var is unset, read `~/.claude/env` directly; do not print its contents.
- Never echo the value, never write it into a Notion page, a repo, a commit message, a log file, or a PR body.
- Never commit `~/.claude/env`.

---

## 2. Always confirm before API-backed actions

Before you perform an action that will actually hit an external service — not just read tool help, not just look up a value — **confirm with the user**. One short sentence is enough:

> About to create a Notion page in `Engineering / Reports` titled "Cross-AZ Investigation" and attach `cross-az-memo.pdf`. OK to proceed?

Read-only calls (`gh pr view`, `notion-search`, `aws s3 ls`) don't need confirmation. Anything that writes, sends, publishes, triggers, or changes state does.

This rule is not optional even when you're "just following the plan we agreed on" — the plan's approval was to write the plan, not to execute each API call within it.

---

## 3. Destructive operations require two-factor approval

A **destructive operation** is anything that:

- Deletes data, resources, branches, files, comments, pages, messages, or rows.
- Overwrites existing state irreversibly (force-push, `terraform destroy`, `aws s3 rm --recursive`, amending a published commit, dropping a DB table).
- Revokes or rotates credentials.
- Sends an outward-facing message, notification, or webhook that can't be unsent (Slack/email/SMS/PagerDuty/calendar invite to external attendees).
- Merges or closes a PR/issue.
- Triggers a workload that costs money or takes production traffic (TFC apply on a prod workspace, CI deploy, Lambda invoke against prod).

### Two-factor protocol

1. **Outline**. In one message, state:
   - *What* the operation is (exact command or API call).
   - *Why* you're doing it.
   - *Blast radius* (one short sentence: which environment, which users, reversibility).
2. **Stop.** Do not chain this confirmation with any other tool calls — not before, not after. The confirmation must be a standalone turn.
3. **Wait** for the user's explicit approval or rejection. Silence is not approval. A previous approval for something similar is not approval.
4. **Execute** only the operation you outlined. If the scope changes, restart the protocol.

Example (good):

> Destructive: delete GitHub branch `feat/abandoned-experiment` on `prototypdigital/mg-infrastructure`.
> Why: PR #1142 was closed without merging; branch is stale.
> Blast radius: local-only impact; branch can be restored from reflog within 90 days via GitHub's "restore branch" UI.
> Proceed? (y/n)

Example (bad — don't do this):

> I'll delete the branch and then update the Notion doc.
> *(tool call: delete branch)*

The second is forbidden because it (a) chained the destructive action with unrelated work and (b) didn't wait for explicit approval.

---

## 4. MCP gaps: falling back to CLI or API

MCP servers sometimes don't expose every operation (Notion MCP can't upload files; Slack MCP can't delete messages; etc.). When you need an operation the MCP doesn't support:

1. **Say so.** Explicitly tell the user: "Notion MCP doesn't support file uploads, falling back to the REST API."
2. **Use the official CLI first** if one exists, **then the API**. Same preference order as §1.
3. **Destructive rule still applies with no exceptions.** If the fallback operation is destructive, you go through the full two-factor protocol in §3, even if the MCP-native equivalent would have been routine.

---

## 5. Credentials hygiene checklist

- [ ] Token comes from `~/.claude/env` or the user's shell env — never hardcoded.
- [ ] CLI session preferred over token when the CLI supports it.
- [ ] No token value ends up in chat output, logs, commits, PRs, or Notion.
- [ ] No local path (`/Users/vlaja/...`, `/tmp/...`) ends up in external systems.
- [ ] `gh` URLs produced by any action are surfaced back as clickable markdown links.

---

## 6. When in doubt

- If you're unsure whether an action is destructive, treat it as destructive.
- If you're unsure whether a token exists, ask before attempting the call.
- If the MCP and CLI disagree on what an operation does, trust the CLI.
