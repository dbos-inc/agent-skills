---
title: Use Queues for Concurrent Workflows
impact: HIGH
impactDescription: Spawned tasks have no durability or flow control
tags: queue, enqueue, concurrency, background
---

## Use Queues for Concurrent Workflows

Register a queue with `dbos.register_queue(name, QueueOptions { .. }, QueueConflict::UpdateIfLatestVersion)` after `launch` (a queue is a database row). Enqueue a workflow by starting it with `StartOptions { queue: Some(QueueRequest::new(name)), .. }`. Enqueued from a workflow, it is a child of that workflow. Queued workflows start in FIFO order.

**Incorrect (tokio::spawn for each task):**

```rust
async fn process_all(dbos: DBOS, tasks: Vec<String>) {
    let process_task = dbos
        .register_workflow("process_task", process_task)
        .unwrap();
    // WRONG: plain tokio tasks have no durability, and no flow control.
    for task in tasks {
        let process_task = process_task.clone();
        tokio::spawn(async move { process_task.run(task).await });
    }
}
```

**Correct (a queue):**

```rust
async fn process_all(
    dbos: &DBOS,
    process_task: &WorkflowRef<String, String>,
    tasks: Vec<String>,
) -> dbos::Result<Vec<String>> {
    // A queue is a database row: register it after launch.
    dbos.register_queue(
        "task_queue",
        QueueOptions::default(),
        QueueConflict::UpdateIfLatestVersion,
    )
    .await?;

    let mut handles = Vec::new();
    for task in tasks {
        let options = StartOptions {
            queue: Some(QueueRequest::new("task_queue")),
            ..StartOptions::default()
        };
        handles.push(process_task.start_with(task, options).await?);
    }
    let mut results = Vec::new();
    for handle in handles {
        results.push(handle.result().await?);
    }
    Ok(results)
}
```

`QueueConflict` says what `register_queue` does when the queue exists: `UpdateIfLatestVersion` (overwrite only from the latest app version; the usual choice at startup), `AlwaysUpdate` or `NeverUpdate`. To enqueue a workflow this process doesn't have, enqueue it by name with `dbos.enqueue(name, queue, input)` or `enqueue_with(name, input, EnqueueOptions { .. })`.
