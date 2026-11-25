---
name: interactive
description: Trigger whenever the user says "interactive", "interactively", "interactive prompt", "interactive picker", or asks for a decision "via UI" / "as a picker". Forces the question to be rendered through the `AskUserQuestion` UI element (chips / native picker) instead of plain chat text, and supports chaining multiple `AskUserQuestion` calls when one answer needs to inform the next question.
---

# Rule: "interactive" means UI, not chat

When the user's message contains "interactive", "interactively", "interactive prompt", "interactive picker", "as a picker", or "via UI" (case-insensitive), every subsequent question you ask in that turn must be rendered through `AskUserQuestion`. No chat-based `1` / `2` / `yes / no` prompts, no numbered lists, no "reply with …" instructions.

This applies to:

- Direct decisions ("which region?", "apply now or defer?").
- Disambiguations ("multiple matches — which one?").
- Confirmations where there's a meaningful choice ("destructive plan — proceed, rerun, or abort?"). Pure y/n confirmations without alternatives stay in chat.

If `AskUserQuestion` is currently deferred (not yet loaded), load it via `ToolSearch` with `select:AskUserQuestion` first, then render.

## Chaining questions for deeper context

When the first answer doesn't give you enough to act, chain follow-ups as additional `AskUserQuestion` calls in sequence — each one is its own UI round.

Patterns:

1. **Sequential narrowing** — ask a broad question, then narrow based on the choice.
   - Round 1: "Where does this go?" → top-level category.
   - Round 2: "Within <chosen category>, which page?" → specific target.

2. **Branching** — the first answer determines whether the second question is even asked.
   - If the user picks "apply now", render round 2 with implementation options.
   - If the user picks "defer", stop — no round 2 needed.

3. **Multi-dimensional** — batch up to 4 unrelated-but-parallel questions in a single `AskUserQuestion` call (it supports 1–4 questions per call). Use this when all dimensions are needed together and none depend on another's answer.

4. **Confirm-then-configure** — round 1 asks yes/no (with alternatives, not bare y/n), round 2 configures the "yes" path.

## Writing good options

- 2–4 options per question (the tool's limit). If you have more, either batch by theme or ask a follow-up round.
- Put the recommended option first and suffix its label with " (Recommended)".
- Labels stay short (1–5 words). Put trade-offs in the `description` field.
- Do not include an explicit "Other" — the tool injects a free-form "Other" automatically.
- Use `preview` blocks only when a visual comparison actually helps (mockups, side-by-side config). Avoid for plain decisions.

## Anti-patterns

- Rendering an `AskUserQuestion` and also restating the options as a numbered list in chat — pick one (the UI).
- Asking "Does this plan look good?" — ambiguous and chat-shaped. Use concrete decision options, or use `ExitPlanMode` if you're in plan mode.
- Chaining more than ~3 rounds for a single decision — if you need that many, you're probably missing context; do research (Read/Grep) between rounds instead of asking again.

## Default for skill-driven asks (no keyword needed)

When **any** skill — including this one — tells you to "ask the user", "confirm with the user", "prompt the user", "pick", "choose", or otherwise surface a decision, render it through `AskUserQuestion` by default. Do not fall back to chat-based numbered options ("reply 1 or 2", "approve (a) or (b)?"). The keyword trigger above is the **mandatory** case; this is the **default** case.

Stay in chat only when the question genuinely has no discrete options:

- Pure yes/no with no meaningful alternative. (If there's a third path — "defer", "abort", "retry" — it's not pure y/n; use UI.)
- Free-form text input ("paste the page URL", "what's the PR number?").
- The invoking skill explicitly says "in chat".

Apply the same design rules (2–4 options, recommended first, short labels, trade-offs in `description`) as for keyword-triggered asks.

## Self-triggered asks (no skill or keyword needed)

Even without a skill or keyword, surface a decision through `AskUserQuestion` when your next action branches on something the user would want to steer. Common triggers:

- **Routing / scoping** — which branch, which PR (existing vs new), which repo, which target directory.
- **Bundling** — include the change in an open PR/commit vs split it out.
- **Destination** — which Notion page, which Slack channel, which issue to comment on.
- **Naming** — slug / branch name / file name where more than one convention is plausible.

Rule of thumb: if you catch yourself about to take an action whose scope the user might disagree with *after the fact*, ask first. Offer 2–3 options with the recommended one first and a one-line trade-off per option.

## When to use plain chat anyway

You may still skip `AskUserQuestion` when the "question" is really a status update or a checkpoint, not a decision — e.g. "here's the diff; tell me when to proceed". But if you catch yourself writing "approve X or Y?" in prose, that's a decision — lift it into `AskUserQuestion`.
