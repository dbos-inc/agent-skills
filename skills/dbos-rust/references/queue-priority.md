---
title: Set Queue Priority for Workflows
impact: HIGH
impactDescription: Urgent work waits behind a backlog
tags: queue, priority, ordering
---

## Set Queue Priority for Workflows

Set `priority` on the `QueueRequest`: 1 to 2,147,483,647, lower runs first, FIFO within a priority. Workflows with no priority run before any with one. `Some(0)` is refused.

**Incorrect (priority 0):**

```rust
async fn urgent_bad(process_task: &WorkflowRef<String, String>) -> dbos::Result<()> {
    // WRONG: priority 0 is refused; priorities run from 1 (highest) upwards.
    let options = StartOptions {
        queue: Some(QueueRequest {
            priority: Some(0),
            ..QueueRequest::new("task_queue")
        }),
        ..StartOptions::default()
    };
    process_task
        .start_with("urgent".to_owned(), options)
        .await?;
    Ok(())
}
```

**Correct (priority 1):**

```rust
async fn urgent(process_task: &WorkflowRef<String, String>) -> dbos::Result<()> {
    // Lower numbers run first. Workflows with no priority run before any with one.
    let options = StartOptions {
        queue: Some(QueueRequest {
            priority: Some(1),
            ..QueueRequest::new("task_queue")
        }),
        ..StartOptions::default()
    };
    process_task
        .start_with("urgent".to_owned(), options)
        .await?;
    Ok(())
}
```
