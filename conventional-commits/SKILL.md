---
name: conventional-commits
description: >
  Commit message standard for all git commits. Use this skill whenever creating a
  commit message or reviewing one. Before executing any git commit, show the user a
  preview — including target branch, added/removed/modified files, and the proposed
  message — and wait for explicit confirmation. Apply even for small or "obvious"
  changes; consistent commit messages make the git history readable and automation
  (changelogs, release notes) reliable.
---

# Conventional Commits

Consistent commit messages make history scannable and enable automated tooling like
changelogs and release notes. The format is simple once internalized.

---

## Preview before every commit

Before running `git commit`, show the user a preview in this format and wait for explicit confirmation:

```
Target branch: <branch-name>

Added files:
  + path/to/file.ext
  (omit section if none)

Removed files:
  - path/to/file.ext
  (omit section if none)

Modified files:
  ~ path/to/file.ext
  (omit section if none)

Commit message:
  <type>(<scope>): <description>

  <body, if any>
```

If the user rejects or suggests changes, adjust and show the preview again before committing.

---

## Format

```
<type>(<scope>): <description>

[optional body]

[optional footer(s)]
```

- `type` and `description` are required; `scope` is optional.
- Title line: max 72 characters.
- Description: **lowercase**, no trailing period, **imperative mood** ("add" not "added", "fix" not "fixed").

---

## Types

| Type | When to use |
|---|---|
| `feat` | New feature for the user |
| `fix` | Bug fix |
| `refactor` | Code change that neither adds a feature nor fixes a bug |
| `test` | Adding or correcting tests |
| `docs` | Documentation only |
| `chore` | Maintenance tasks, configs, dependencies |
| `perf` | Performance improvement |
| `ci` | CI/CD pipeline changes |
| `build` | Build system or external dependency changes |
| `revert` | Reverts a previous commit |
| `style` | Formatting, whitespace — no logic change |

---

## Breaking changes

Two accepted formats:

**1. Exclamation mark after the type:**
```
feat!: remove support for legacy /v1/events endpoint
```

**2. `BREAKING CHANGE` footer:**
```
feat(api): change odds response format

BREAKING CHANGE: field `odd` renamed to `oddDecimal` across all endpoints.
```

---

## Body and footers

- Separate title from body with a **blank line**.
- The body explains the **why**, not the what — the diff already shows what changed.
- Footers follow the format `Token: value` or `Token #reference`.

```
fix(crawler): handle null odds returned by Betano API

Null odds from the API caused a NullPointerException in the mapper.
Added a defensive null check before the Double conversion.

Closes #42
```

---

## Valid examples

```
feat(crawler): add Hiperbet support via Altenar platform
fix(service): prevent division by zero in margin calculation
refactor(json): extract AltenarJsonMapper into dedicated class
test(crawler): add real JSON fixture for Betano
chore: upgrade Playwright dependency to 1.42
docs: document crawler complexity levels
feat!: remove discontinued /api/v1/partidas endpoint
```

## Invalid examples

```
fix: Bug fixed                         ← uppercase and past tense
feat: add new feature.                 ← trailing period
update crawler                         ← missing type
feat: add Hiperbet support via Altenar platform and fix null odds parsing in JSON mapper that caused NullPointerException  ← title too long
```
