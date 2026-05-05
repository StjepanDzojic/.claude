---
name: automation-suggestions
description: Governs when and how to proactively suggest creating new hooks, subagents, or skills. Use when you notice a repeated permission-accepted pattern across sessions (hook candidate), a frequently-performed operation (subagent candidate), or a cross-cutting concern that would benefit multiple subagents (skill candidate). Every suggestion requires explicit user approval before anything is written.
---

# Automation Suggestions

> **Interactive asks.** When this skill tells you to ask the user anything with discrete options (approve / modify / decline a suggested hook, subagent, skill, or script), render it via `AskUserQuestion` per the `interactive` skill — never as chat-based numbered options or bare `(y/n)` prose when there's a real third path.

This skill tells you when and how to suggest new automation. It does **not** let you create hooks, subagents, or skills on your own — every suggestion needs explicit user approval before you touch `~/.claude/`.

---

## How to detect repetition across sessions

The harness writes every session transcript to `~/.claude/projects/<project-slug>/*.jsonl`. Each file contains every tool call, every permission prompt, and every permission decision. You can read these files with Grep/Bash to find patterns **without keeping your own counter** — they are the source of truth.

Useful greps:

- Permission prompts that were approved: search for `"permission_request"` / `"permission_decision":"accept"` pairs in jsonl entries.
- Tool usage: `"tool_name":"<name>"` across the project's `*.jsonl` files.
- Per-agent or per-skill triggers: `"subagent_type":"..."` and `Skill` invocations.

Keep analysis bounded: look at roughly the last 10 sessions (or last 30 days) of the current project, not the full history. Cross-project patterns only matter if the behavior is clearly global (e.g. `gh pr create`, not project-specific Terraform commands).

You do not need to keep a running counter in memory. Run the analysis on demand when you notice a pattern, or when the user asks.

---

## 1. Hook candidate — repeated permission-accepted operations

Trigger the suggestion when **all** of the following hold:

- The same tool call shape (same command, same flags pattern, or same MCP operation) appears **≥ 5 times** in the recent jsonl history, across at least **2 distinct sessions**.
- Every single occurrence was gated by a permission prompt, and every single occurrence was **accepted**.
- The operation is side-effect-free or has a narrow, predictable blast radius (e.g. read-only `gh pr view`, `terraform fmt -recursive`, `which <tool>`). Never suggest hooks for destructive ops.
- A **zero-token hook** is technically possible — i.e. it can run as a deterministic shell command configured in `settings.json` without needing the model in the loop.

If any of those fails, do not suggest a hook.

### Suggestion format (standalone turn — do not chain)

> **Hook candidate detected.**
>
> I've been prompted for and you've approved `<exact tool call shape>` N times across M sessions. It's side-effect-free and can run as a zero-token hook.
>
> - **Trigger**: `<PreToolUse | PostToolUse | UserPromptSubmit | ...>`
> - **Matcher**: `<regex or tool filter>`
> - **Command**: `<deterministic shell command>`
> - **What it replaces**: the permission prompt + my tool call.
> - **Rollback**: delete the hook entry from `settings.json`; nothing else changes.
>
> Want me to add this hook to `~/.claude/settings.json`? (y/n)
>
> If yes, I'll hand off to the `update-config` skill to write it.

Never write the hook yourself. Hand off to `update-config` on approval.

---

## 2. Subagent candidate — frequently-performed operation

Trigger the suggestion when:

- The same **multi-step operation** (not a single tool call — a *workflow*) appears **≥ 3 times** across the recent history.
- The steps are broadly the same each time: same order, same intent, similar inputs.
- The workflow is self-contained enough to be expressed as a cold-start prompt (a new agent with no conversation context could do it from a single instruction).
- It doesn't already live in an existing subagent.

Count the workflow, not individual tool calls. "`gh pr create` ran 20 times" is a hook candidate. "User scaffolds a new Terraform module → writes files → runs fmt → opens PR → labels it — 4 times" is a subagent candidate.

### Suggestion format (standalone turn)

> **Subagent candidate detected.**
>
> You've done the workflow "<short name>" N times. It would fit a dedicated global subagent at `~/.claude/agents/<slug>.md`.
>
> - **Scope**: <one sentence describing responsibility>
> - **Inputs**: <what the user provides>
> - **Outputs**: <what the subagent produces>
> - **Why subagent, not skill**: it's a single clean responsibility, not cross-cutting; it owns an end-to-end flow rather than being reused by other agents.
> - **Why subagent, not hook**: the workflow needs judgment (the model must decide what to write, not just run a fixed command).
>
> Want me to draft the subagent file for your review? (y/n)

Never write the subagent file without approval. On approval, draft it into a `diff` or a review-only file path; only move into `~/.claude/agents/` after the user confirms the draft.

---

## 3. Skill candidate — cross-cutting concern

A **skill** is right when the concern is used by multiple subagents (or by the main agent + one subagent) and the rules are the same across all of them.

Trigger the suggestion when:

- A behavior, guardrail, or procedure applies identically to **≥ 2 existing or proposed subagents** (or to the main agent plus ≥ 1 subagent).
- Centralizing it avoids copy-paste drift — if the rule changes, you'd otherwise have to edit N agent files.
- The concern is policy / procedure / knowledge (how to do X safely), not a workflow (do X end-to-end).

### Required reasoning step — do NOT skip

Before suggesting the skill, explicitly enumerate:

1. **Every subagent that would be affected** (existing or proposed, by name).
2. For each affected subagent, state: *does the concern resolve identically, or does this subagent have a variation?*
3. If any subagent has a meaningful variation:
   - Can the skill capture the variation with a parameter or a clearly-labelled section (e.g. "destructive-op protocol differs for read-only services")?
   - Or is it better to (a) split into two skills, (b) keep the variation in the subagent itself, or (c) leave the subagents separate and not share a skill at all?
4. Pick the smallest packaging that still avoids drift. One shared skill is only the right answer when the concern is truly uniform.

### Suggestion format (standalone turn)

> **Skill candidate detected.**
>
> The concern "<short name>" is showing up in:
> - `<subagent-a>` — resolves identically.
> - `<subagent-b>` — resolves identically.
> - `<subagent-c>` — variation: <description>. Proposed handling: <parameter | section | split | leave separate>.
>
> **Why a skill, not a subagent**: it's a policy/procedure reused across agents, not a single responsibility to own.
> **Why a skill, not a hook**: requires model judgment (context-dependent decisions, not a deterministic command).
> **Affected files if created**: `~/.claude/skills/<slug>.md` (new), and lightweight "see `<slug>` skill" pointers in each affected subagent (replacing the duplicated content).
>
> My recommendation: **<one shared skill | split into two | variation stays in subagent C>**.
>
> Want me to draft the skill? (y/n)

---

## 4. Script candidate — determinism fix

Trigger the suggestion when **both** of the following hold:

- The same operation has produced **materially different output across runs** for the same (or equivalent) input — small ordering differences, missing files, stray log lines, inconsistent naming, different ruleset/permissions on identical-looking resources. In other words, **model-level randomness** is leaking into work that needs to be identical every time.
- The operation **can be pinned down deterministically in bash**: it's a sequence of CLI calls, file writes, and substitutions, not a judgment call the model must make fresh each time.

Canonical example: `~/.claude/agents/scaffold-action-repo.md` delegates to `~/Dev/Projects/github-actions/.github/scripts/scaffold-action-repo.sh`. Every new `prototypdigital/action-*` repo needs to be byte-for-byte identical in structure, workflows, and ruleset. The model can't guarantee that on its own, so the agent is a thin wrapper and a bash script owns the actual scaffolding.

The marker for a script candidate is: *"we need this identical every time and the model can't be trusted to reproduce it."* Not *"this is repetitive"* (that's a subagent) and not *"this is a deterministic one-liner"* (that's a hook).

### Where the script lives

Two valid locations, pick by scope:

- **Repo-local** (preferred when tied to one codebase): `<repo>/.github/scripts/<verb>-<noun>.sh` or `<repo>/scripts/<verb>-<noun>.sh`. Lives with the code it operates on, reviewable in PRs, CI can exercise it.
- **Global** (when the operation spans repos / machine state): `~/.claude/scripts/<verb>-<noun>.sh`. Use sparingly — global scripts are invisible to collaborators.

Prefer repo-local whenever the operation is bounded to one project. Only use `~/.claude/scripts/` for operations that are inherently per-machine (e.g. "sync all my terraform-modules clones").

### Script conventions

Every suggested script must:

- Start with `#!/usr/bin/env bash` and `set -euo pipefail`.
- Take inputs as positional args or env vars — never read from stdin unprompted.
- Be idempotent where possible (safe to re-run).
- Emit human-readable progress to stderr and machine-usable output (if any) to stdout.
- Have no TODOs, no `# adjust this` comments, no "usually you'd want" hedging. Deterministic means deterministic.
- Exit non-zero on any unexpected state — don't paper over.

### Suggestion format (standalone turn)

> **Script candidate detected.**
>
> The operation "<short name>" produced different output across runs: <point to 2+ concrete divergences you saw, with session refs>. The variance is model-level randomness — the steps themselves are deterministic.
>
> - **Proposed location**: `<repo>/.github/scripts/<slug>.sh` **or** `~/.claude/scripts/<slug>.sh` (pick based on scope).
> - **Inputs**: <args / env vars>.
> - **What it does** (bullet list of deterministic steps).
> - **Which subagent(s)/skill(s) will call it**: <names>, replacing inline steps with a single `bash <path> <args>` call.
> - **Rollback**: delete the script; subagents that referenced it are easy to restore from git.
>
> Want me to draft the script and the subagent update together? (y/n)
>
> If yes, I'll produce both as a review-only diff first. Nothing lands in `~/.claude/` or a repo without approval.

### Hand-off back to the subagent/skill

When a script is approved and written, the caller (subagent or skill) should shrink to:

1. Validate inputs.
2. `bash <path-to-script> <args>`.
3. Post-process / report.

Never have the subagent *duplicate* the script's logic "for safety" — that reintroduces the randomness.

---

## 5. Rules that apply to all four

- **One suggestion per turn.** Do not stack hook + subagent + skill suggestions in the same message. If multiple patterns trip at once, pick the highest-leverage one and queue the others for later.
- **Standalone turn.** Never chain a suggestion with an unrelated tool call — the user's yes/no is the whole point of the turn.
- **Never create without approval.** This applies even if the user previously said "go ahead and do this kind of thing automatically" — each new automation is approved individually. An old blanket approval isn't approval for a new file in `~/.claude/`.
- **Be honest about uncertainty.** If the pattern only happened 3 times and you're suggesting anyway, say so. If you can't tell whether two subagents' variations are really identical, say so and ask.
- **Cite the evidence.** Name specific sessions/files in the jsonl history that support the count. The user should be able to verify.
- **Prefer silence over noisy suggestions.** Only suggest when the signal is clear. Drive-by "you could automate this someday" suggestions are noise.

---

## 6. Hand-off to the right creation tool

Once approved:

- **Hook** → invoke the `update-config` skill to write the hook into `settings.json`.
- **Skill** → invoke `anthropic-skills:skill-creator` if the user prefers, or draft the `.md` file directly.
- **Subagent** → draft the `.md` file in `~/.claude/agents/<slug>.md` following the same frontmatter + body format as existing agents (`scaffold-action-repo`, `doc-template-manager`, etc.).
- **Script** → draft the bash file at the chosen location with `chmod +x`, then update the calling subagent/skill to delegate to it in a single commit.

Never short-circuit the approval. The suggestion and the creation are two separate turns.
