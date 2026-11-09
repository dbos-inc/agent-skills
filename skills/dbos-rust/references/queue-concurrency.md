---
title: Control Queue Concurrency
impact: HIGH
impactDescription: Unbounded queues exhaust memory, connections or downstream APIs
tags: queue, concurrency, worker, limits
---

## Control Queue Concurrency

`worker_concurrency` limits how many of a queue's workflows one process runs at once; `concurrency` limits them across all processes. Prefer `worker_concurrency`: every `PENDING` workflow on the queue counts toward `concurrency`, including old versions'. `worker_concurrency: Some(0)` pauses the queue. The per-partition limits must be at least 1, `worker_concurrency` can't be below `partition_worker_concurrency`, and `concurrency` can't be below `worker_concurrency` or either per-partition concurrency limit. So to pause a queue that sets `partition_worker_concurrency`, clear it and set `worker_concurrency: Some(0)` in the same `QueueChange` (clearing the queue's only partition limit also un-partitions it), and `concurrency: Some(0)` works only when `worker_concurrency` is 0 or unset and no per-partition concurrency limit is set. Change limits at runtime with `update_queue` and a `QueueChange` of `Change::Set(..)` / `Change::Leave` fields.

**Incorrect (no limits):**

```rust
async fn unbounded(dbos: &DBOS) -> dbos::Result<()> {
    // No limits: every enqueued workflow starts as soon as it's dequeued,
    // which can exhaust memory or an external API.
    dbos.register_queue(
        "task_queue",
        QueueOptions::default(),
        QueueConflict::UpdateIfLatestVersion,
    )
    .await?;
    Ok(())
}
```

**Correct (limits, changed at runtime):**

```rust
async fn register_limited(dbos: &DBOS) -> dbos::Result<()> {
    let options = QueueOptions {
        // At most 5 at once in each process...
        worker_concurrency: Some(5),
        // ...and at most 20 across all processes.
        concurrency: Some(20),
        ..QueueOptions::default()
    };
    dbos.register_queue("task_queue", options, QueueConflict::UpdateIfLatestVersion)
        .await?;

    // Change a limit at runtime, without a restart.
    let change = QueueChange {
        worker_concurrency: Change::Set(Some(10)),
        ..QueueChange::default()
    };
    dbos.update_queue("task_queue", change).await?;
    Ok(())
}
```

`queue(name)` and `list_queues()` read the stored configuration. A startup `register_queue` with `UpdateIfLatestVersion` (on the latest version) or `AlwaysUpdate` overwrites changes made at runtime; register with `NeverUpdate` to keep them.
