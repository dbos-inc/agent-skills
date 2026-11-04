---
title: Cancel, Resume, Fork, and Rewind Workflows
impact: MEDIUM
impactDescription: Management calls from a workflow body must be recorded, or recovery repeats them
tags: workflow, cancel, resume, fork, rewind, delete, management
---

## Cancel, Resume, Fork, and Rewind Workflows

Outside a workflow, manage workflows with `DBOS` methods: `cancel`/`cancel_all`, `resume`/`resume_with`/`resume_all`, `fork`/`fork_with`/`fork_all`, `rewind`/`rewind_with`, `delete`/`delete_all`, `set_workflow_delay`. Inside a workflow, use the free functions (`dbos::cancel_workflow`, `resume_workflow`, `fork_workflow`, `rewind_workflow`, `delete_workflow`, ...), which are recorded as steps.

**Incorrect (capturing DBOS to cancel a child):**

```rust
fn register_supervisor(dbos: &DBOS) -> dbos::Result<()> {
    let instance = dbos.clone();
    // WRONG: the closure holds the DBOS instance, which owns the closure.
    dbos.register_workflow("supervisor", move |child_id: String| {
        let dbos = instance.clone();
        async move {
            dbos.cancel(&child_id).await?;
            Ok::<_, dbos::Error>(())
        }
    })?;
    Ok(())
}
```

**Correct (free functions in the workflow body):**

```rust
async fn supervisor(child_id: String) -> dbos::Result<()> {
    // Inside a workflow, use the free functions: each is recorded as a step.
    dbos::cancel_workflow(&child_id).await?;
    let handle = dbos::resume_workflow::<(), dbos::EngineOnly>(&child_id).await?;
    handle.result().await?;
    Ok(())
}
```

Outside a workflow:

```rust
async fn cancel_example(dbos: &DBOS, workflow_id: &str) -> dbos::Result<()> {
    dbos.cancel(workflow_id).await?;
    Ok(())
}

async fn cancel_tree(dbos: &DBOS, workflow_ids: &[&str]) -> dbos::Result<()> {
    // Cancels each workflow and all its descendants.
    let cancelled = dbos.cancel_all(workflow_ids, Children::Include).await?;
    Ok(())
}
```

```rust
async fn fork_example(dbos: &DBOS, workflow_id: &str) -> dbos::Result<()> {
    // Fork from step 3, onto the version of the code that has the fix.
    let options = ForkOptions {
        app_version: Some("1.2.1"),
        ..ForkOptions::default()
    };
    let handle = dbos
        .fork_with::<String, dbos::EngineOnly>(Fork::new(workflow_id, 3), options)
        .await?;
    println!("Forked {} as {}", workflow_id, handle.workflow_id());
    Ok(())
}
```

```rust
async fn rewind_example(dbos: &DBOS, workflow_id: &str) -> dbos::Result<()> {
    // Re-execute the workflow from step 3, keeping its workflow ID.
    let options = RewindOptions {
        start_step: 3,
        ..RewindOptions::default()
    };
    let handle = dbos
        .rewind_with::<String, dbos::EngineOnly>(workflow_id, options)
        .await?;
    let result = handle.result().await?;
    Ok(())
}
```

`resume`, `fork` and `rewind` take the workflow's result and error types as type parameters. Rewind only a finished workflow (cancel a running one first); it keeps the ID, discards steps from the chosen step on, and re-enqueues it. `Children::Include` on the bulk forms reaches every descendant. Deleting a running workflow makes it fail at its next step: cancel first. Cancelling takes the workflow from the running execution without interrupting it: a step in progress runs to its end, its result isn't recorded, and it runs again if the workflow is resumed. Awaiting a cancelled workflow returns `Error::AwaitedWorkflowCancelled` inside a workflow, `Error::WorkflowCancelled` outside one.
