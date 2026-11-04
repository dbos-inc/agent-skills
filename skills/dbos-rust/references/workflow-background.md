---
title: Start Workflows in Background
impact: CRITICAL
impactDescription: Awaiting every workflow inline blocks callers and gives up concurrency
tags: workflow, start, run, handle, background
---

## Start Workflows in Background

`register_workflow` returns a typed `WorkflowRef<P, R, E>`. `start(input)` starts the workflow in the background and returns a `WorkflowHandle`; `run(input)` starts it and waits for its result. Await `handle.result()` when the result is needed. Called inside a workflow, both start a child workflow, recorded in the parent so recovery doesn't start it twice.

**Incorrect (waiting on each workflow before starting the next):**

```rust
async fn start_example(process_order: &WorkflowRef<Order, String>) -> dbos::Result<()> {
    let order = Order {
        id: 1,
        item: "widget".to_owned(),
    };

    // Start the workflow in the background and get a handle to it...
    let handle = process_order.start(order).await?;
    let result = handle.result().await?;
    println!("Workflow result: {result}");

    // ...or run it and wait for its result in one call.
    let result = process_order
        .run(Order {
            id: 2,
            item: "gadget".to_owned(),
        })
        .await?;
    Ok(())
}
```

**Correct (start in the background, collect results later):**

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

`retrieve_workflow::<R, E>(id)` (free function in a workflow, `DBOS` method outside) returns a handle to an existing workflow. `handle.status()` returns its current status.
