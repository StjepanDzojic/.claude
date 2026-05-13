---
name: pr-review
description: Reviews a GitHub pull request for code quality, bugs, improvements, and security issues. Trigger whenever the user asks to review a PR, check a pull request, audit changes, or says things like "check the pr", "review this pr", "any issues with this pr", "is this pr good". Accepts a PR URL, PR number, or no argument (lists open PRs). Always use this skill for any PR review task.
---

# PR Review

## Input

The user will provide a PR URL (e.g. `https://github.com/owner/repo/pull/34`), a bare PR number, or nothing.

- **URL**: extract repo (`owner/repo`) and number from the URL, run `gh pr view <N> --repo owner/repo` and `gh pr diff <N> --repo owner/repo`
- **Number only**: run `gh pr view <N>` and `gh pr diff <N>` against the current repo
- **No argument**: run `gh pr list` and ask the user which PR to review

Suppress non-fatal `gh` stderr with `2>/dev/null` — GraphQL deprecation warnings and Dependabot notices are noise.

## Deep context

After fetching the diff, read the full source of any changed files that are central to the PR (not generated files, lock files, or large fixtures). Skimming a diff without seeing how the changed code fits into its surroundings misses bugs and architectural issues.

## Review structure

Always produce exactly these sections — skip a section only if it genuinely has nothing to say (note that you skipped it and why):

### Overview
What the PR does in 2–4 sentences. Include the flow if relevant (e.g. URL routing changes).

### Code quality
Style consistency, naming, dead code, unnecessary complexity, and adherence to project conventions visible in the surrounding codebase.

### Bugs / correctness
Logic errors, edge cases, off-by-one, null/undefined paths, race conditions, type mismatches. For each issue: quote the relevant line(s), explain the failure mode, suggest a fix.

### Improvements
Things that work but could be cleaner, simpler, or more idiomatic. Distinguish clearly from bugs — these are non-blocking suggestions.

### Security
Input validation, injection vectors (SQL, XSS, command), secrets in code, insecure defaults, overly broad permissions, exposed error details. If nothing is concerning, say so explicitly rather than omitting the section.

## Tone

- Be direct and specific. Quote code, cite file paths and line numbers (format: `file.tsx:42`).
- Distinguish severity: **bug** (must fix), **improvement** (non-blocking), **nitpick** (minor style).
- Don't pad with praise for ordinary correct code. Reserve positive remarks for genuinely non-obvious good choices.
- If the PR is clean, say so concisely — a short review on a clean PR is a good outcome.
