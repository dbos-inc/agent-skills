---
title: Deduplicate Queued Workflows
impact: HIGH
impactDescription: Duplicate enqueues run the same work twice
tags: queue, deduplication, idempotency, singleton
---

## Deduplicate Queued Workflows

Set `deduplication_id` on the `QueueRequest`: while a workflow with that ID is enqueued or running on the queue, another enqueue with it fails with `dbos::sysdb::Error::QueueDeduplicated`. Set `duplication_policy: DuplicationPolicy::ReturnExisting` to get a handle to the holder instead (its input wins).

**Incorrect (no deduplication):**

```rust
async fn start_report(report: &WorkflowRef<String, String>, user_id: &str) -> dbos::Result<()> {
    // WRONG: a double-click starts two reports for the same user.
    let options = StartOptions {
        queue: Some(QueueRequest::new("reports")),
        ..StartOptions::default()
    };
    report.start_with(user_id.to_owned(), options).await?;
    Ok(())
}
```

**Correct (one per user at a time):**

```rust
async fn start_report_once(
    report: &WorkflowRef<String, String>,
    user_id: &str,
) -> dbos::Result<WorkflowHandle<String>> {
    let options = StartOptions {
        queue: Some(QueueRequest {
            // At most one report per user enqueued or running at once...
            deduplication_id: Some(user_id),
            // ...and a second request gets a handle to the first.
            duplication_policy: DuplicationPolicy::ReturnExisting,
            ..QueueRequest::new("reports")
        }),
        ..StartOptions::default()
    };
    report.start_with(user_id.to_owned(), options).await
}
```

```rust
async fn dedup_example(
    task_workflow: &WorkflowRef<String, String>,
    user_id: &str,
) -> dbos::Result<()> {
    let mut queue = QueueRequest::new("example_queue");
    queue.deduplication_id = Some(user_id);
    let options = StartOptions {
        queue: Some(queue),
        ..StartOptions::default()
    };
    match task_workflow.start_with("task".to_owned(), options).await {
        Ok(handle) => println!("Enqueued {}", handle.workflow_id()),
        Err(dbos::Error::SystemDatabase(dbos::sysdb::Error::QueueDeduplicated { .. })) => {
            // A workflow for this user is already enqueued or running.
        }
        Err(error) => return Err(error),
    }
    Ok(())
}
```
