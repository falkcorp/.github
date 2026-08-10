<!-- file: instructions/commit-messages.md -->
<!-- version: 1.0.0 -->
<!-- guid: 7e64f098-516b-4019-ba02-999219cdf438 -->
<!-- last-edited: 2026-08-10 -->

# Commit Message Standards

All commits use [Conventional Commits](https://www.conventionalcommits.org/) format.

## Format

```text
type(scope): short description

Optional body — explain WHY, not what.

Co-Authored-By: ...
```

## Types

| Type | Use for |
|---|---|
| `feat` | New feature |
| `fix` | Bug fix |
| `refactor` | Code change that neither fixes a bug nor adds a feature |
| `test` | Adding or updating tests |
| `docs` | Documentation only |
| `chore` | Build process, dependency updates, housekeeping |
| `perf` | Performance improvement |
| `ci` | CI/CD changes |

## Rules

- Subject line: imperative mood, lowercase, no period, max 72 chars.
- Scope is the affected package or area: `feat(dedup):`, `fix(server):`, `chore(ci):`.
- Breaking changes: add `!` after scope (`feat(api)!:`) and explain in body.
- Reference issues in the body or footer: `Closes #123`, `Fixes #456`.
