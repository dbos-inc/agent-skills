---
title: Partition Queues for Per-Key Ordering and Tenant Isolation
impact: MEDIUM
impactDescription: Enforces per-tenant concurrency or sequential FIFO execution per key without head-of-line blocking across keys
tags: queue, partition, partition_key, partition_concurrency, multi-tenant, ordering
---

## Partition Queues for Per-Key Ordering and Tenant Isolation

In DBOS Rust, there is no boolean `partition_queue: true` flag: setting **any** per-partition limit on `QueueOptions` (`partition_concurrency`, `partition_worker_concurrency`, or `partition_rate_limit`) makes the queue partitioned (`Queue::is_partitioned() == true`). Every workflow enqueued (or forked/rewound) onto a partitioned queue **must** supply a non-empty `partition_key`, or no worker will ever dequeue it.

**Incorrect (enqueuing onto a partitioned queue without a partition_key):**

```rust
use dbos::{DBOS, Enqueue, QueueConflict, QueueOptions, StartOptions, WorkflowRef};

async fn stranded_enqueue(
    dbos: &DBOS,
    process_event: &WorkflowRef<String, ()>,
) -> dbos::Result<()> {
    // Setting `partition_concurrency: Some(1)` partitions the queue
    dbos.register_queue(
        "user-events",
        QueueOptions {
            partition_concurrency: Some(1),
            ..QueueOptions::default()
        },
        QueueConflict::UpdateIfLatestVersion,
    )
    .await?;

    // BUG: `partition_key` is `None`! Partitioned queues sweep `WHERE queue_partition_key IS NOT NULL`,
    // so this row sits in `ENQUEUED` forever and is never picked up!
    process_event
        .start_with(
            "login".to_owned(),
            StartOptions {
                queue: Some(Enqueue::new("user-events")),
                ..StartOptions::default()
            },
        )
        .await?;
    Ok(())
}
```

**Correct (sequential per-user ordering with partition_concurrency: Some(1) and partition_key):**

```rust
use dbos::{DBOS, Enqueue, QueueConflict, QueueOptions, StartOptions, WorkflowRef};

async fn setup_and_enqueue_partitioned(
    dbos: &DBOS,
    process_event: &WorkflowRef<String, ()>,
    user_id: &str,
    payload: String,
) -> dbos::Result<()> {
    // `partition_concurrency: Some(1)` runs each user's events strictly one-at-a-time in FIFO order,
    // while up to 50 different users make progress concurrently across the fleet
    dbos.register_queue(
        "user-events",
        QueueOptions {
            concurrency: Some(50),
            worker_concurrency: Some(10),
            partition_concurrency: Some(1),
            ..QueueOptions::default()
        },
        QueueConflict::UpdateIfLatestVersion,
    )
    .await?;

    process_event
        .start_with(
            payload,
            StartOptions {
                queue: Some(Enqueue {
                    partition_key: Some(user_id),
                    ..Enqueue::new("user-events")
                }),
                ..StartOptions::default()
            },
        )
        .await?;
    Ok(())
}
```

### Partitioning Gotchas

- **Empty `partition_key` is rejected**: `partition_key: Some("")` fails at submission with `Error::InvalidArgument`.
- **Forks and rewinds do not inherit `partition_key`**: When calling `fork_with` or `rewind_with` targeting a partitioned `queue_name`, explicitly pass `queue_partition_key: Some(key)` on `ForkOptions` or `RewindOptions`.
- **Do not partition a non-empty unpartitioned queue in place**: Adding a partition limit via `update_queue` while unkeyed rows are already `ENQUEUED` strands those unkeyed rows; drain the queue first before enabling partition limits.

Reference: [dbos::QueueOptions::partition_concurrency](https://docs.rs/dbos/latest/dbos/struct.QueueOptions.html#structfield.partition_concurrency), [dbos::Enqueue::partition_key](https://docs.rs/dbos/latest/dbos/struct.Enqueue.html#structfield.partition_key)
