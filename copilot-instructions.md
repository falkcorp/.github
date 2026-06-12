# falkcorp — Org-wide Coding Standards

These instructions apply to all repositories in the falkcorp org.
Per-repo `CLAUDE.md` and `.github/copilot-instructions.md` add project-specific context on top.

## File Version Headers (MANDATORY — read this first)

**Every file you create or modify must have a version header updated.** Copilot and other reviewers
will flag PRs where modified files are missing or have stale headers. See full rules in
[instructions/file-headers.md](instructions/file-headers.md).

**Quick reference:**

```go
// Go files — first lines, before `package`:
// file: internal/package/filename.go
// version: 1.2.3
// guid: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
// last-edited: YYYY-MM-DD
```

```markdown
<!-- All other files (Markdown, YAML, JSON, HTML): -->
<!-- file: path/to/file.md -->
<!-- version: 1.2.3 -->
<!-- guid: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx -->
<!-- last-edited: YYYY-MM-DD -->
```

Bump the version and set `last-edited` to today on **every** change, no matter how small.

## Git Operations

1. **MCP GitHub tools** (preferred) — use for all git operations when available.
2. **Native git** (fallback) — when MCP tools aren't available.

All commits MUST use conventional commit format: `type(scope): description`.
Full rules: [instructions/commit-messages.md](instructions/commit-messages.md).

## Language-specific standards

- Go: [instructions/go.md](instructions/go.md)
- TypeScript/React: [instructions/typescript.md](instructions/typescript.md)
- File headers: [instructions/file-headers.md](instructions/file-headers.md)
- Commit messages: [instructions/commit-messages.md](instructions/commit-messages.md)

## Security

- Never commit secrets, tokens, or credentials.
- Validate all user input at system boundaries.
- Use parameterized queries — no string interpolation in SQL.
- Pin GitHub Action references to SHAs, not tags.

## Testing

- Every new function with business logic needs a test.
- Use table-driven tests for Go, Vitest for TypeScript.
- Integration tests that hit real I/O must be skippable in short/unit mode.
