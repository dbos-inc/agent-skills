---
title: Partition Queues for Per-Entity Limits
impact: HIGH
impactDescription: A global limit serializes unrelated entities behind each other
tags: queue, partition, per-user, tenant
---

## Partition Queues for Per-Entity Limits

Setting `partition_concurrency`, `partition_worker_concurrency` or `partition_rate_limit` partitions a queue: the limits apply per partition key. Every enqueue on a partitioned queue must set `partition_key`; one without it is never dequeued. Queue-wide limits on a partitioned queue are enforced too.

**Incorrect (a global limit of 1):**

```rust
async fn per_user_bad(dbos: &DBOS) -> dbos::Result<()> {
    // WRONG: a global concurrency of 1 serializes every user behind each other.
    let options = QueueOptions {
        concurrency: Some(1),
        ..QueueOptions::default()
    };
    dbos.register_queue("per_user", options, QueueConflict::UpdateIfLatestVersion)
        .await?;
    Ok(())
}
```

**Correct (one per user):**

```rust
async fn per_user(dbos: &DBOS, process_task: &WorkflowRef<String, String>) -> dbos::Result<()> {
    // One workflow at a time per user; different users run concurrently.
    let options = QueueOptions {
        partition_concurrency: Some(1),
        ..QueueOptions::default()
    };
    dbos.register_queue("per_user", options, QueueConflict::UpdateIfLatestVersion)
        .await?;

    // Every enqueue on a partitioned queue must name its partition.
    let options = StartOptions {
        queue: Some(QueueRequest {
            partition_key: Some("user-123"),
            ..QueueRequest::new("per_user")
        }),
        ..StartOptions::default()
    };
    process_task.start_with("task".to_owned(), options).await?;
    Ok(())
}
```

```rust
fn combined_limits() -> QueueOptions {
    QueueOptions {
        // At most 2 workflows per tenant at once...
        partition_concurrency: Some(2),
        // ...and at most 20 across all tenants.
        concurrency: Some(20),
        ..QueueOptions::default()
    }
}
```
