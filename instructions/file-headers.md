# File Version Headers

**Every file you create or modify MUST have a version header at the top. This is mandatory — missing or stale headers cause CI review failures.**

## When to update

- **Any time you touch a file**: bump the version and set `last-edited` to today's date.
- Use semantic versioning: patch (typos/fixes), minor (new content), major (structural changes).

## Formats by file type

### Go files (`.go`)

```go
// file: internal/package/filename.go
// version: 1.2.3
// guid: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
// last-edited: 2026-06-12
```

Place this block as the very first lines of the file, before `package`.

### Markdown, YAML, JSON, HTML, and all other text files

```markdown
<!-- file: path/to/file.md -->
<!-- version: 1.2.3 -->
<!-- guid: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx -->
<!-- last-edited: 2026-06-12 -->
```

### Shell scripts and Python files (`.sh`, `.py`)

```bash
# file: scripts/deploy.sh
# version: 1.2.3
# guid: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
# last-edited: 2026-06-12
```

## GUIDs

- Every file gets a unique GUID that never changes (it identifies the file even after renames).
- For new files: generate a random UUID v4.
- For existing files that lack a GUID: add one and treat this as a patch bump.

## Common mistakes to avoid

- **Do not skip the header update when making "small" changes** — every change bumps the version.
- **Do not use HTML comment format (`<!-- -->`) in Go files** — Go uses `//` comments.
- **Do not use Go comment format (`//`) in Markdown files** — use `<!-- -->`.
- **The `last-edited` field is a date, not a timestamp** — use `YYYY-MM-DD` only.
