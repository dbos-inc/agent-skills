# dbos-python

> **Note:** `CLAUDE.md` is a symlink to this file.

## Overview

DBOS Python SDK (3.x) for building reliable, fault-tolerant applications with durable workflows. Use this skill when writing Python code with DBOS, creating workflows and steps, using queues, datasource transactions, or schedules, using DBOSClient from external applications, upgrading DBOS Python 2.x code to 3.x, or building applications that need to be resilient to failures.

## Structure

```
dbos-python/
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
| 1 | Lifecycle | CRITICAL | `lifecycle-` |
| 2 | Workflow | CRITICAL | `workflow-` |
| 3 | Step | HIGH | `step-` |
| 4 | Queue | HIGH | `queue-` |
| 5 | Communication | MEDIUM | `comm-` |
| 6 | Pattern | MEDIUM | `pattern-` |
| 7 | Testing | LOW-MEDIUM | `test-` |
| 8 | Client | MEDIUM | `client-` |
| 9 | Advanced | LOW | `advanced-` |

Reference files are named `{prefix}-{topic}.md` (e.g., `query-missing-indexes.md`).

## Available References

**Advanced** (`advanced-`):
- `references/advanced-async.md`
- `references/advanced-patching.md`
- `references/advanced-serialization.md`
- `references/advanced-shared-database.md`
- `references/advanced-upgrading-v3.md`
- `references/advanced-versioning.md`

**Client** (`client-`):
- `references/client-enqueue.md`
- `references/client-setup.md`

**Communication** (`comm-`):
- `references/comm-events.md`
- `references/comm-messages.md`
- `references/comm-streaming.md`

**Lifecycle** (`lifecycle-`):
- `references/lifecycle-config.md`
- `references/lifecycle-fastapi.md`

**Pattern** (`pattern-`):
- `references/pattern-classes.md`
- `references/pattern-debouncing.md`
- `references/pattern-idempotency.md`
- `references/pattern-scheduled.md`
- `references/pattern-sleep.md`

**Queue** (`queue-`):
- `references/queue-basics.md`
- `references/queue-concurrency.md`
- `references/queue-deduplication.md`
- `references/queue-delay.md`
- `references/queue-listening.md`
- `references/queue-management.md`
- `references/queue-partitioning.md`
- `references/queue-rate-limiting.md`

**Step** (`step-`):
- `references/step-basics.md`
- `references/step-retries.md`
- `references/step-transactions.md`

**Testing** (`test-`):
- `references/test-fixtures.md`

**Workflow** (`workflow-`):
- `references/workflow-attributes.md`
- `references/workflow-background.md`
- `references/workflow-constraints.md`
- `references/workflow-control.md`
- `references/workflow-determinism.md`
- `references/workflow-introspection.md`
- `references/workflow-timeout.md`

---

*37 reference files across 9 categories*