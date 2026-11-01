---
title: Control Per-Worker and Fleet-Wide Queue Concurrency
impact: HIGH
impactDescription: Bounds simultaneous workflow executions locally per process and globally across replicas
tags: queue, concurrency, worker_concurrency, throttling, pause
---

## Control Per-Worker and Fleet-Wide Queue Concurrency

`QueueOptions` separates per-process limits from fleet-wide limits: `worker_concurrency` caps how many workflows from the queue run simultaneously **in one `DBOS` process** (tracked in memory with zero extra SQL cost), whereas `concurrency` caps how many workflows from the queue are `PENDING` **across every replica combined** (enforced in PostgreSQL at `REPEATABLE READ`).

**Incorrect (setting worker_concurrency higher than global concurrency):**

```rust
use dbos::{DBOS, QueueConflict, QueueOptions};

async fn invalid_limits(dbos: &DBOS) -> dbos::Result<()> {
    // Rejected with `Error::InvalidArgument`: a narrower scope (`worker_concurrency = 20`)
    // may never exceed its enclosing scope (`concurrency = 5`)!
    dbos.register_queue(
        "llm-calls",
        QueueOptions {
            worker_concurrency: Some(20),
            concurrency: Some(5),
            ..QueueOptions::default()
        },
        QueueConflict::UpdateIfLatestVersion,
    )
    .await?;
    Ok(())
}
```

**Correct (combining worker_concurrency and fleet-wide concurrency within hierarchy rules):**

```rust
use dbos::{Change, DBOS, QueueChange, QueueConflict, QueueOptions};

async fn configure_and_pause_queue(dbos: &DBOS) -> dbos::Result<()> {
    // Up to 5 workflows per replica, and at most 20 across all replicas combined
    dbos.register_queue(
        "llm-calls",
        QueueOptions {
            worker_concurrency: Some(5),
            concurrency: Some(20),
            ..QueueOptions::default()
        },
        QueueConflict::UpdateIfLatestVersion,
    )
    .await?;

    // Pause dequeuing across every replica without restarting (`Some(0)` holds back new claims)
    dbos.update_queue(
        "llm-calls",
        QueueChange {
            worker_concurrency: Change::Set(Some(0)),
            ..QueueChange::default()
        },
    )
    .await?;

    Ok(())
}
```

### Concurrency Hierarchy and Pausing Rules

- **Prefer `worker_concurrency` when protecting per-process resources** (CPU, memory, local connection pool): it is checked locally without raising transaction isolation. Add `concurrency` when protecting a shared downstream system (a database, third-party API, or GPU cluster) across N replicas.
- **Containment invariants**: Whenever two limits are both `Some`, the narrower limit cannot exceed the wider one:
  - `worker_concurrency <= concurrency`
  - `partition_concurrency <= concurrency`
  - `partition_worker_concurrency <= worker_concurrency`
  - `partition_worker_concurrency <= partition_concurrency`
- **Pausing a queue with `Some(0)`**: Setting `worker_concurrency: Some(0)` pauses dequeuing on every replica (and works on partitioned queues too). Setting `concurrency: Some(0)` also pauses an unpartitioned queue, provided `worker_concurrency` is `None` or `Some(0)` (`partition_concurrency` must be `>= 1` when set, so always pause partitioned queues via `worker_concurrency: Some(0)`).

Reference: [dbos::QueueOptions](https://docs.rs/dbos/latest/dbos/struct.QueueOptions.html), [dbos::Queue](https://docs.rs/dbos/latest/dbos/struct.Queue.html)
