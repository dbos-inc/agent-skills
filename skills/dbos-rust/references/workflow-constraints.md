---
title: Follow Workflow Constraints
impact: CRITICAL
impactDescription: Calls that need a workflow fail inside a step, and recovery depends on the workflow body
tags: workflow, constraints, step, rules
---

## Follow Workflow Constraints

A step is a leaf: only its result is recorded. Starting or enqueueing a workflow, `dbos::recv`, `dbos::set_event`, `dbos::patch` and a debouncer's `debounce` inside a step fail with `Error::InsideStep`. Make those calls from the workflow body. Calls like `dbos::send`, `get_event`, `write_stream` and `list_workflows` run plainly inside a step, as part of it, so they happen again if the step is retried.

**Incorrect (workflow operations inside a step):**

```rust
async fn my_workflow(other: WorkflowRef<String, String>) -> dbos::Result<String> {
    let data = step("fetch_and_start", || async {
        let data = fetch_data().await?;
        // WRONG: starting a workflow inside a step fails with Error::InsideStep.
        other.start(data.clone()).await?;
        // WRONG: recv and set_event inside a step fail with Error::InsideStep.
        dbos::set_event("status", &"fetched").await?;
        Ok(data)
    })
    .await?;
    Ok(data)
}
```

**Correct (workflow operations from the workflow body):**

```rust
async fn my_workflow(other: WorkflowRef<String, String>) -> dbos::Result<String> {
    // Steps only do external operations.
    let data = step("fetch_data", fetch_data).await?;
    // Start child workflows, receive messages and set events from the workflow body.
    let handle = other.start(data.clone()).await?;
    let msg: Option<String> = dbos::recv(Some("topic"), Duration::from_secs(60)).await?;
    dbos::set_event("status", &"done").await?;
    handle.result().await
}
```

- Register every workflow with `register_workflow` before `launch`; register queues and create schedules after it.
- A workflow takes exactly one input; use a struct or tuple for several, `()` for none.
- Do NOT mutate shared state in memory from workflows or steps.
- A panic in a workflow does not crash the process: the workflow stays `PENDING`, like a crash, and is recovered on the next launch.
- Outside a workflow, `dbos::step` and `dbos::sleep` run as plain calls; `set_event`, `recv` and `send` fail with `Error::NotInWorkflow`.
