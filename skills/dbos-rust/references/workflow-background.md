---
title: Start Workflows in Background
impact: CRITICAL
impactDescription: Awaiting every workflow inline blocks callers and gives up concurrency
tags: workflow, start, run, handle, background
---

## Start Workflows in Background

`register_workflow` returns a typed `WorkflowRef<P, R, E>`. `start(input)` starts the workflow in the background and returns a `WorkflowHandle`; `run(input)` starts it and waits for its result. Await `handle.result()` when the result is needed. Called inside a workflow, both start a child workflow, recorded in the parent so recovery doesn't start it twice.

**Incorrect (running each workflow to completion in the request):**

```rust
async fn handle_request(process_task: &WorkflowRef<String, String>, task: String) -> dbos::Result<String> {
    // WRONG: the request waits for the whole workflow; a slow workflow times out the request.
    process_task.run(task).await
}
```

**Correct (start in the background, return the ID):**

```rust
async fn handle_request(
    process_task: &WorkflowRef<String, String>,
    task: String,
) -> dbos::Result<String> {
    // Start in the background and return the ID at once; the workflow runs durably.
    let handle = process_task.start(task).await?;
    Ok(handle.workflow_id().to_owned())
}
```

`retrieve_workflow::<R, E>(id)` (free function in a workflow, `DBOS` method outside) returns a handle to an existing workflow. `handle.status()` returns its current status.
