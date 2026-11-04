---
title: Set Workflow Timeouts
impact: CRITICAL
impactDescription: Workflows without timeouts can run forever
tags: workflow, timeout, deadline, cancellation
---

## Set Workflow Timeouts

Set a durable timeout with the `timeout` field of `StartOptions` (or `RunOptions`), a `Timeout`: `Inherit` (default: a child takes its parent's deadline), `None`, or `Explicit(duration)` (`duration.into()`). On expiry the workflow is cancelled, with the children that inherited its deadline (children with their own timeout or `Timeout::None` keep running). In the process running the workflow, it is dropped at its current `.await`; awaiting its handle returns `Error::WorkflowCancelled`. Timeouts are start-to-completion (an enqueued workflow's clock starts when it is dequeued) and survive restarts.

**Incorrect (tokio::time::timeout around a workflow):**

```rust
// WRONG: dropping the future stops waiting, but the workflow keeps running.
let result = tokio::time::timeout(Duration::from_secs(60), workflow.run(input)).await;
```

**Correct (a durable timeout):**

```rust
async fn timeout_example(example_workflow: &WorkflowRef<String, String>) -> dbos::Result<()> {
    let options = StartOptions {
        timeout: Duration::from_secs(12 * 60 * 60).into(),
        ..StartOptions::default()
    };
    let handle = example_workflow
        .start_with("wait-for-cancel".to_owned(), options)
        .await?;
    Ok(())
}
```

In the process running the workflow, a timeout stops it at once, including a step in progress, which records nothing and runs again if the workflow is resumed. Do NOT wrap `run` in `tokio::time::timeout`: dropping the future doesn't cancel the workflow, which keeps running durably.
