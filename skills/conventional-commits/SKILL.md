---
name: conventional-commits
description: "Write commit messages in Conventional Commits format (`type(scope): description`). Use whenever creating, amending, squashing, or rewriting a commit message — including requests like \"commit this\", \"commit my changes\", \"save my work\", \"split this into commits\", or \"clean up these commit messages\" — even when the user doesn't mention any convention. Also use when reviewing or normalizing commit log style. Not for other git mechanics such as branching, rebasing conflicts, remotes, hooks, or signing."
---

# Conventional Commits

Default every commit message to [Conventional Commits](https://www.conventionalcommits.org/). The structured prefix makes history scannable and lets tooling derive changelogs and version bumps automatically (`feat` → minor, `fix` → patch, `!`/breaking footer → major) — that payoff only works if every message follows the same shape.

## Format

```
<type>(<scope>)!: <subject>

<body>

<footer(s)>
```

Only the type and subject are required. Everything else is optional — don't add empty sections.

## Choosing the type

| Type | Use for |
| --- | --- |
| `feat` | New user-facing capability |
| `fix` | Bug fix |
| `docs` | Documentation-only changes |
| `refactor` | Code change that neither fixes a bug nor adds behavior |
| `perf` | Performance improvement |
| `test` | Adding or correcting tests |
| `build` | Build system or dependency changes (make, cargo, npm, …) |
| `ci` | CI/CD pipeline configuration |
| `chore` | Maintenance touching no source or test files |
| `style` | Formatting, whitespace — no change to code meaning |
| `revert` | Reverting an earlier commit |

Quick disambiguation: user-visible behavior → `feat`; something broken made to work → `fix`; internal reshuffle → `refactor`; otherwise pick the closest fit and use `chore` as fallback. When one commit bundles mixed work, type it after its dominant change — or suggest splitting it if the parts are cleanly separable.

## Subject, scope, body

- **Subject**: imperative mood ("add", never "added" or "adds"), lowercase first letter, no trailing period, aim for ≤ 72 characters. A reader of `git log --oneline` should understand the change from the subject alone.
- **Scope** (optional): a short noun for the affected area — module, package, or directory name (`feat(auth):`, `fix(parser):`). Omit when no single area applies.
- **Body** (optional): explains *why* the change is needed or what context a future reader lacks; wrap lines at 72 characters.
- **Breaking changes**: add `!` immediately before the colon **and** a `BREAKING CHANGE: <what broke and how to migrate>` footer. The bare `!` isn't enough for most changelog tooling, so include both.

## Respect stronger signals

Before writing, glance at `git log --oneline -10`. If the repo consistently follows a different deliberate style (e.g. Linux kernel `subsystem: summary` prefixes), match the repo instead — commit conventions exist for the consumers of that particular history, and established repo precedent outranks this default. Use Conventional Commits when history is empty, inconsistent, or already conventional.

## Examples

"add dark mode toggle to settings page" →

```
feat(settings): add dark mode toggle
```

"rewrote the storage layer, old API methods are gone" →

```
feat(storage)!: replace callback API with async iterators

BREAKING CHANGE: Storage.read()/write() removed. Migrate to
Storage.iter() and Storage.stream(); see docs/migration.md.
```

"typo in README installation section" →

```
docs: fix typo in installation instructions
```
