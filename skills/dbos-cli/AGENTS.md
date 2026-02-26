# dbos-cli

> **Note:** `CLAUDE.md` is a symlink to this file.

## Overview

>

## Structure

```
dbos-cli/
  SKILL.md       # Main skill file - read this first
  AGENTS.md      # This navigation guide
  CLAUDE.md      # Symlink to AGENTS.md
  references/    # Detailed reference files
```

## Usage

1. Read `SKILL.md` for the main skill instructions
2. Browse `references/` for detailed documentation on specific topics
3. Reference files are loaded on-demand - read only what you need

## Reference Categories

| Priority | Category | Impact | Prefix |
|----------|----------|--------|--------|
| 1 | Migration | CRITICAL | `migrate-` |
| 2 | Configuration | HIGH | `config-` |
| 3 | Initialization | MEDIUM | `init-` |
| 4 | Workflow Management | MEDIUM | `workflow-` |
| 5 | Local Development | LOW-MEDIUM | `local-` |

Reference files are named `{prefix}-{topic}.md` (e.g., `query-missing-indexes.md`).

## Available References

**Configuration** (`config-`):
- `references/config-database.md`
- `references/config-migrate-commands.md`

**Initialization** (`init-`):
- `references/init-project.md`

**Local Development** (`local-`):
- `references/local-postgres.md`
- `references/local-reset.md`

**Migration** (`migrate-`):
- `references/migrate-commands.md`

**Workflow Management** (`workflow-`):
- `references/workflow-management.md`

---

*7 reference files across 5 categories*