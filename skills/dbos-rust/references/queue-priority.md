---
title: Order Queue Dequeues with Explicit Priorities
impact: LOW-MEDIUM
impactDescription: Ensures critical queued workflows dequeue ahead of lower-priority batch work without starving unprioritized defaults
tags: queue, priority, Enqueue, ordering, fifo
---

## Order Queue Dequeues with Explicit Priorities

Every queue in DBOS Rust orders waiting workflows by `(priority ASC, created_at ASC)` automatically, with no extra `priority_enabled` flag on `QueueOptions`. Lower numbers run **first**: explicit priorities span `Some(1)..=Some(i32::MAX as u32)`, while `None` stores `0` (unprioritized), which sorts **ahead** of every explicit `Some(1..)` priority.

**Incorrect (passing Some(0) or u32 > i32::MAX, or leaving batch work at priority: None):**

```rust
use dbos::{Enqueue, StartOptions, WorkflowRef};

async fn bad_priority_usage(
    job_wf: &WorkflowRef<String, ()>,
    urgent: String,
    backfill: String,
) -> dbos::Result<()> {
    // Fails with `Error::InvalidArgument`: `Some(0)` is forbidden (`1..=2_147_483_647` only)!
    job_wf
        .start_with(
            urgent,
            StartOptions {
                queue: Some(Enqueue {
                    priority: Some(0),
                    ..Enqueue::new("jobs")
                }),
                ..StartOptions::default()
            },
        )
        .await?;

    // Trap: Leaving low-priority backfill at `priority: None` stores `0` in Postgres,
    // which dequeues BEFORE `priority: Some(1)` urgent work!
    job_wf
        .start_with(
            backfill,
            StartOptions {
                queue: Some(Enqueue::new("jobs")),
                ..StartOptions::default()
            },
        )
        .await?;
    Ok(())
}
```

**Correct (assigning explicit numeric tiers across all prioritized producers):**

```rust
use dbos::{Enqueue, StartOptions, WorkflowRef};

const PRIORITY_CRITICAL: u32 = 1;
const PRIORITY_STANDARD: u32 = 10;
const PRIORITY_BACKFILL: u32 = 100;

async fn enqueue_with_priority(
    job_wf: &WorkflowRef<String, ()>,
    payload: String,
    priority: u32,
) -> dbos::Result<()> {
    job_wf
        .start_with(
            payload,
            StartOptions {
                queue: Some(Enqueue {
                    priority: Some(priority),
                    ..Enqueue::new("jobs")
                }),
                ..StartOptions::default()
            },
        )
        .await?;
    Ok(())
}

async fn submit_mixed_workload(job_wf: &WorkflowRef<String, ()>) -> dbos::Result<()> {
    enqueue_with_priority(job_wf, "archive-logs".to_owned(), PRIORITY_BACKFILL).await?;
    enqueue_with_priority(job_wf, "sync-profile".to_owned(), PRIORITY_STANDARD).await?;
    // Dequeues ahead of both `10` and `100` (ties within `1` break by FIFO `created_at`)
    enqueue_with_priority(job_wf, "fraud-alert".to_owned(), PRIORITY_CRITICAL).await?;
    Ok(())
}
```

### Priority Rules

- **Valid range**: `Some(1)` through `Some(i32::MAX as u32)` (`2_147_483_647`). Both `Some(0)` and values `> i32::MAX as u32` fail at submission with `Error::InvalidArgument`.
- **Assign explicit priorities consistently**: Because `priority: None` writes `0` in PostgreSQL (`0 < 1`), mixing `None` and `Some(1)` on the same queue causes `None` jobs to run ahead of `Some(1)`. Once a queue uses priorities, give every producer on that queue an explicit `Some(...)` tier.

Reference: [dbos::Enqueue::priority](https://docs.rs/dbos/latest/dbos/struct.Enqueue.html#structfield.priority)
