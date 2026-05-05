---
name: semantic-commits
description: Use whenever you write a git commit message, craft a PR title, or set up commit-parsing automation. Enforces Conventional Commits (feat/fix/chore/refactor/docs/test/ci/build/perf, optional scope, `!` for breaking) on ALL prototypdigital and vlaja repos — regardless of whether the repo uses release-please today. Migration to release-please is ongoing, so consistent commit hygiene everywhere is the prerequisite.
---

# Semantic Commits

Every commit and every PR title on every repo follows [Conventional Commits](https://www.conventionalcommits.org/).

## Why

1. **Release-please migration is in progress.** Repos that already use release-please (e.g. `terraform-modules`) parse commit titles to generate changelogs and pick the next version bump. Repos that don't use it yet will — enforcing the format everywhere means adopting release-please later is a config change, not a history rewrite.
2. **Grep-able history.** `git log --oneline | grep '^feat'` gives you a real feature list. Unstructured messages don't.
3. **PR titles double as commit titles** when squash-merging. Consistent titles → consistent release notes.
4. **Renovate and other bots** default to semantic commit output; matching their format makes their PRs indistinguishable from human PRs in the changelog.

## Format

```
<type>(<scope>)<!>: <subject>

<optional body>

<optional footer(s)>
```

### Types (standard set — nothing else)

| Type | When |
|---|---|
| `feat` | New user-visible capability. Minor bump. |
| `fix` | Bug fix. Patch bump. |
| `chore` | Maintenance that doesn't change behavior (deps, tooling, config). No bump. |
| `refactor` | Internal restructuring; no behavior change. No bump. |
| `docs` | Documentation only. No bump. |
| `test` | Test-only changes. No bump. |
| `ci` | CI/CD pipeline changes. No bump. |
| `build` | Build system / packaging. No bump. |
| `perf` | Performance change with no other behavior change. Patch bump. |

If none fit, the change is probably two changes — split it.

### Scope

Optional, lowercase, hyphen-separated. Use it when the change touches a single discrete unit:

- In `terraform-modules`: the module name — `feat(vpc):`, `fix(rds-instance):`, `feat(craftcms-db-backup)!:`.
- In `mg-infrastructure`: the environment or the subsystem — `feat(acceptance):`, `chore(ci):`.
- In app repos: the feature area — `feat(auth):`, `fix(checkout):`.

Omit scope if the change spans multiple units or isn't naturally attributable to one.

### Breaking changes

Append `!` before the colon: `feat(vpc)!: drop support for IPv4-only subnets`.

Also add a `BREAKING CHANGE:` footer explaining the break and the migration path. Release-please reads both the `!` and the footer; the `!` alone triggers a major bump, the footer goes into the changelog verbatim.

### Subject

- Imperative mood ("add", not "added" or "adds").
- Lowercase first letter.
- No trailing period.
- Under ~72 chars.

### Body / footer

Optional. Use the body for the *why*. Use footers for machine-readable metadata: `Closes #123`, `BREAKING CHANGE: …`, `Refs: …`.

## PR titles

The PR title must itself be a valid Conventional Commit. When you squash-merge, GitHub uses the PR title as the squash commit — a bad PR title pollutes release-please output.

## When release-please is present

Detect with: file `release-please-config.json` / `.release-please-manifest.json` at the repo root, or the signature commit `chore: update versions and changelog [skip ci]` in recent history.

When present:
- **Don't apply `bump:*` labels manually** — release-please infers the bump from the commit type (`feat` → minor, `fix`/`perf` → patch, `!` / `BREAKING CHANGE:` → major).
- **Don't hand-edit `CHANGELOG.md`** — release-please owns it.
- **Don't hand-bump versions** in `package.json`, `VERSION`, or similar — release-please opens a release PR that does it.

## When release-please is absent

Still use the Conventional Commit format. Version bumps and changelog updates happen manually (or via the repo's own convention, e.g. `bump:minor` label on `terraform-modules` PRs before the release-please migration finishes). The format being consistent now means the eventual switchover is lossless.

## Examples

```
feat(vpc): add flow logs to private subnets
fix(rds-instance): filter subnets by Name tag pattern
feat(craftcms-db-backup)!: configurable cpu/memory, defaults bumped to 1024/2048

BREAKING CHANGE: default cpu/memory raised from 512/1024 to 1024/2048. Consumers relying on the old defaults must pin explicitly.
chore: bump aws provider to 5.80.0
docs: clarify rds-proxy reader target group usage
ci: switch validate-pr to OIDC for AWS auth
```

## What not to do

- `Update things` — no type, no scope, no subject discipline.
- `feat: added new feature` — past tense, vague subject.
- `FEAT(VPC):` — uppercase.
- `feat(vpc): new flow logs.` — trailing period.
- `wip` / `fixes` / `.` — never, not even temporarily; rebase before PR to clean up.
- `feat: new module and also refactor old one` — two changes, split the commit.
