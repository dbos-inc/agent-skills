---
title: Inspect, Reconfigure, and Drain Queues at Runtime
impact: LOW-MEDIUM
impactDescription: Tunes live queue limits without restarts and avoids stranding enqueued workflows on queue deletion
tags: queue, management, update_queue, delete_queue, list_queues, QueueChange, Change
---

## Inspect, Reconfigure, and Drain Queues at Runtime

Because DBOS Rust stores queue definitions in PostgreSQL and re-reads them on every dequeue sweep, changes applied through `update_queue` (on `DBOS` or `Client`) take effect across the entire fleet within one polling interval, with zero restarts. Inspect queues with `dbos.queue(name)` (not `get_queue`) and `dbos.list_queues()`, and drain or cancel waiting workflows before calling `dbos.delete_queue(name)`.

**Incorrect (deleting an active queue without draining or confusing None with Leave):**

```rust
use dbos::{Change, DBOS, QueueChange};

async fn dangerous_queue_ops(dbos: &DBOS) -> dbos::Result<()> {
    // Mistake 1: `Change::Set(None)` CLEARS `rate_limit` to NULL (unthrottled);
    // use `Change::Leave` (via `..QueueChange::default()`) to keep existing columns untouched!
    dbos.update_queue(
        "orders",
        QueueChange {
            worker_concurrency: Change::Set(Some(8)),
            rate_limit: Change::Set(None),
            ..QueueChange::default()
        },
    )
    .await?;

    // Mistake 2: `delete_queue` removes ONLY the metadata row in `dbos.queues`!
    // Any workflows still `ENQUEUED` or `DELAYED` on `"orders"` become stranded with no worker!
    dbos.delete_queue("orders").await?;
    Ok(())
}
```

**Correct (partial updates with Change::Leave and cancelling backlog before delete_queue):**

```rust
use std::time::Duration;
use dbos::{Change, Children, DBOS, Queue, QueueChange};
use dbos::sysdb::types::{WorkflowFilter, WorkflowStatus};

async fn tune_queue_live(dbos: &DBOS, name: &str) -> dbos::Result<Option<Queue>> {
    // Update `worker_concurrency` and `polling_interval`, leaving `concurrency` and `rate_limit` intact
    let updated = dbos
        .update_queue(
            name,
            QueueChange {
                worker_concurrency: Change::Set(Some(12)),
                polling_interval: Change::Set(Duration::from_millis(250)),
                ..QueueChange::default()
            },
        )
        .await?;
    let _ = updated;

    // Read a single queue back (`dbos.queue`, NOT `get_queue`)
    dbos.queue(name).await
}

async fn drain_and_retire_queue(dbos: &DBOS, name: &str) -> dbos::Result<usize> {
    // 1. Pause dequeuing so workers stop claiming new items while we cancel the backlog
    dbos.update_queue(
        name,
        QueueChange {
            worker_concurrency: Change::Set(Some(0)),
            ..QueueChange::default()
        },
    )
    .await?;

    // 2. Find and cancel waiting (`ENQUEUED` / `DELAYED`) workflows on this queue
    let waiting = dbos
        .list_workflows(&WorkflowFilter {
            queue_names: vec![name],
            status: vec![WorkflowStatus::Enqueued, WorkflowStatus::Delayed],
            load_input: false,
            load_output: false,
            ..WorkflowFilter::default()
        })
        .await?;
    let ids: Vec<&str> = waiting.iter().map(|w| w.workflow_id.as_str()).collect();
    let cancelled = if ids.is_empty() {
        Vec::new()
    } else {
        dbos.cancel_all(&ids, Children::Skip).await?
    };

    // 3. Remove the queue definition once drained (idempotent if already absent)
    dbos.delete_queue(name).await?;
    Ok(cancelled.len())
}
```

### Queue Management Reference

- **`Change<T>` three-state semantics**: Every field on `QueueChange` defaults to `Change::Leave` (do not touch the column). For nullable limits (`Change<Option<i32>>`, `Change<Option<RateLimit>>`), `Change::Set(Some(v))` sets the limit and `Change::Set(None)` clears the limit back to `NULL`.
- **Listing and ownership**: `dbos.list_queues().await?` returns all queues owned by this `app_name` plus unclaimed (`application_name IS NULL`) queues, ordered alphabetically by name. Attempting to `update_queue` or `delete_queue` a queue owned by another application fails with `RegisteredByAnother`.

Reference: [dbos::DBOS::queue](https://docs.rs/dbos/latest/dbos/struct.DBOS.html#method.queue), [dbos::DBOS::update_queue](https://docs.rs/dbos/latest/dbos/struct.DBOS.html#method.update_queue), [dbos::QueueChange](https://docs.rs/dbos/latest/dbos/struct.QueueChange.html), [dbos::Change](https://docs.rs/dbos/latest/dbos/sysdb/types/enum.Change.html)
