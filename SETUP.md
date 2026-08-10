<!-- file: SETUP.md -->
<!-- version: 1.0.0 -->
<!-- guid: 02124888-a749-4e21-8232-caa7623dc6b7 -->
<!-- last-edited: 2026-08-10 -->

# Standards Setup

## How this repo works

- **GitHub Copilot** reads `copilot-instructions.md` automatically for all falkcorp repos.
- **Claude Code** reads from `~/.claude/standards/` when it's a full clone of this repo.
- **Per-repo CLAUDE.md** stays minimal (project-specific only); global standards come from `~/.claude/CLAUDE.md` which references this clone.

## One-time setup (per machine)

```bash
# Full clone — no shallow clones, no submodule skipping
git clone --recurse-submodules https://github.com/falkcorp/.github ~/.claude/standards
```

Then add to `~/.claude/CLAUDE.md`:

```markdown
## Shared Coding Standards
Standards are in ~/.claude/standards/. Key files:
- File headers (MANDATORY): @~/.claude/standards/instructions/file-headers.md
- Go: @~/.claude/standards/instructions/go.md
- TypeScript: @~/.claude/standards/instructions/typescript.md
- Commits: @~/.claude/standards/instructions/commit-messages.md
```

## Keeping standards up to date

```bash
git -C ~/.claude/standards pull --recurse-submodules
```

Add to crontab for automatic updates:

```cron
0 9 * * 1 git -C ~/.claude/standards pull --recurse-submodules
```

## Shared templates

Document templates that apply across repos live in `templates/`. Reference them
from a consuming repo rather than copying them in — a copied template drifts,
and the copies are what this repo exists to prevent.

- **`templates/executive-summary.md`** — stakeholder-facing write-ups that
  justify what work cost, for a reader who decides about it but does not read
  code. Covers both shapes (single change, and period roundup), the rules that
  make them worth reading, and the naming and update conventions.

A consuming repo should keep `docs/executive-summaries/README.md` as a short
pointer at this file, not a second copy of it.

## Per-repo files

After setup, per-repo files become minimal stubs:

**`.github/copilot-instructions.md`** — repo-specific additions only (org-level applies automatically):

```markdown
<!-- Org-level standards: https://github.com/falkcorp/.github -->
<!-- Project context: see CLAUDE.md -->

# [Repo name] — Additional Context

[Only repo-specific rules here]
```

**`AGENTS.md`** — pointer to CLAUDE.md:

```markdown
See CLAUDE.md for all agent instructions and project context.
```
