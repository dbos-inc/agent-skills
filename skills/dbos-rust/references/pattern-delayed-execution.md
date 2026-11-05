---
title: Delay Workflow Execution
impact: MEDIUM
impactDescription: A workflow that sleeps first holds a worker slot for the whole delay
tags: pattern, delay, queue, delayed
---

## Delay Workflow Execution

Set `delay` on the `QueueRequest` to hold an enqueued workflow `DELAYED` until the delay passes; it takes no worker slot while it waits. Change a waiting workflow's delay with `set_workflow_delay(id, WorkflowDelay::For(d))` or `WorkflowDelay::Until(t)`.

**Incorrect (sleeping at the top of a queued workflow):**

```rust
async fn send_later(task: String) -> dbos::Result<()> {
    // Holds a worker slot on its queue for the whole hour.
    dbos::sleep(Duration::from_secs(60 * 60)).await?;
    step("send", || send(&task)).await
}
```

**Correct (a delayed enqueue):**

```rust
async fn delay_example(process_task: &WorkflowRef<String, String>) -> dbos::Result<()> {
    let options = StartOptions {
        queue: Some(QueueRequest {
            delay: Some(Duration::from_secs(60 * 60)),
            ..QueueRequest::new("example_queue")
        }),
        ..StartOptions::default()
    };
    let handle = process_task.start_with("task".to_owned(), options).await?;
    Ok(())
}
```

```rust
async fn bring_forward(dbos: &DBOS, workflow_id: &str) -> dbos::Result<()> {
    // Start the delayed workflow now.
    dbos.set_workflow_delay(workflow_id, dbos::WorkflowDelay::For(Duration::ZERO))
        .await?;
    Ok(())
}
```
