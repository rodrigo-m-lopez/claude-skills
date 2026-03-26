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
- **Language: always write the description and body in Portuguese**, regardless of the language used in the request or conversation. The `type` and `scope` fields remain in English as they are part of the format convention.

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
feat(crawler): adiciona suporte ao site Hiperbet via plataforma Altenar
fix(service): evita divisão por zero no cálculo de margem
refactor(json): extrai AltenarJsonMapper para classe dedicada
test(crawler): adiciona fixture JSON real da Betano
chore: atualiza dependência do Playwright para 1.42
docs: documenta níveis de complexidade de crawlers
feat!: remove endpoint /api/v1/partidas descontinuado
```

## Invalid examples

```
fix: Bug corrigido                     ← uppercase and past tense
feat: adiciona nova funcionalidade.    ← trailing period
update crawler                         ← missing type
feat: adiciona suporte ao site Hiperbet via Altenar e corrige parsing de odds nulas no mapper JSON que causava NullPointerException  ← title too long
feat(crawler): add Hiperbet support    ← description in English
```
