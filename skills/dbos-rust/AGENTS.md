# dbos-rust

> **Note:** `CLAUDE.md` is a symlink to this file.

## Overview

DBOS Rust SDK for building reliable, fault-tolerant applications with durable workflows on PostgreSQL. Use this skill when writing Rust code with the dbos crate, creating workflows and steps, using queues or cron schedules, integrating DBOS with Axum, using dbos::Client from external applications, or building async Rust applications that need crash recovery and durable execution.

## Structure

```
dbos-rust/
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
- `references/advanced-interops.md`
- `references/advanced-patching.md`
- `references/advanced-serialization.md`
- `references/advanced-shared-database.md`
- `references/advanced-versioning.md`

**Client** (`client-`):
- `references/client-enqueue.md`
- `references/client-setup.md`

**Communication** (`comm-`):
- `references/comm-events.md`
- `references/comm-messages.md`
- `references/comm-streaming.md`

**Lifecycle** (`lifecycle-`):
- `references/lifecycle-axum.md`
- `references/lifecycle-config.md`

**Pattern** (`pattern-`):
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
- `references/queue-priority.md`
- `references/queue-rate-limiting.md`

**Step** (`step-`):
- `references/step-basics.md`
- `references/step-concurrency.md`
- `references/step-retries.md`
- `references/step-transactions.md`

**Testing** (`test-`):
- `references/test-setup.md`

**Workflow** (`workflow-`):
- `references/workflow-attributes.md`
- `references/workflow-background.md`
- `references/workflow-concurrency.md`
- `references/workflow-constraints.md`
- `references/workflow-control.md`
- `references/workflow-determinism.md`
- `references/workflow-instances.md`
- `references/workflow-introspection.md`
- `references/workflow-registration.md`
- `references/workflow-timeout.md`

---

*40 reference files across 9 categories*