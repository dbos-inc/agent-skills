---
title: Start Background and Child Workflows with WorkflowHandle
impact: HIGH
impactDescription: Enables fire-and-forget workflows, checkpointed child orchestration, and cross-process handle retrieval
tags: workflow, background, child, start, start_with, WorkflowHandle, retrieve_workflow
---

## Start Background and Child Workflows with WorkflowHandle

Use `WorkflowRef::start` (or `start_with`) to launch a workflow in the background and receive a `WorkflowHandle<R, E>` immediately without blocking on completion. Dropping a `WorkflowHandle` never cancels the workflow; the workflow continues running durably, and any process can reconnect to it later by ID via `dbos.retrieve_workflow`.

**Incorrect (using tokio::spawn to background a workflow or calling result() twice):**

```rust
use dbos::WorkflowRef;

async fn kick_off_export(export_wf: WorkflowRef<String, String>) {
    // Unnecessary detached task: if the process crashes before `run` inserts the row,
    // the caller never obtained a workflow ID to track!
    tokio::spawn(async move {
        let _ = export_wf.run("report-2026".to_owned()).await;
    });
}
```

**Correct (starting durably with WorkflowRef::start, orchestrating children, and retrieving by ID):**

```rust
use std::time::Duration;
use dbos::{DBOS, EngineOnly, Error, StartOptions, WorkflowHandle, WorkflowRef};
use serde::{Deserialize, Serialize};

#[derive(Debug, thiserror::Error, Serialize, Deserialize)]
#[error("fulfillment failed: {0}")]
struct OrderError(String);

// Parent workflow captures the child's `WorkflowRef` (never `DBOS`) at registration time
async fn fulfill_order(
    ship_child: WorkflowRef<String, String>,
    order_id: String,
) -> dbos::Result<String, OrderError> {
    // Recorded as step `DBOS.start_workflow`; default child ID is `<parent_id>-<step_id>`
    let handle: WorkflowHandle<String> = ship_child.start(order_id.clone()).lift().await?;

    let invoice = dbos::step("create_invoice", || async {
        Ok(format!("inv-{order_id}"))
    })
    .await?;

    // Recorded as step `DBOS.get_result`; `result(self)` consumes the handle once
    match handle.result().lift().await {
        Ok(tracking) => Ok(format!("{invoice}:{tracking}")),
        Err(Error::AwaitedWorkflowCancelled { workflow_id }) => {
            Err(OrderError(format!("shipment {workflow_id} was cancelled")).into())
        }
        Err(other) => Err(other),
    }
}

async fn poll_from_http_handler(dbos: &DBOS, workflow_id: &str) -> dbos::Result<String> {
    // Synchronous handle construction (does no I/O until `status()` or `result()` is polled)
    let handle = dbos.retrieve_workflow::<String, EngineOnly>(workflow_id)?;
    let _current_status = handle.status().await?;

    // Bound `retrieve_workflow` waits outside workflows if the ID came from untrusted input
    tokio::time::timeout(Duration::from_secs(5), handle.result())
        .await
        .map_err(|_| Error::InvalidArgument {
            operation: "await_workflow".into(),
            detail: "still running".to_owned(),
        })?
}
```

### Child Workflow and Handle Semantics

- **Parent-child linkage and deterministic child IDs**: Calling `start`, `start_with`, `run`, or `run_with` from inside a workflow body sets `parent_workflow_id` on the child row and defaults an omitted `StartOptions::workflow_id` to `"{parent_workflow_id}-{step_id}"`. Replaying a crashed parent re-attaches to the existing child instead of spawning a duplicate.
- **Unifying error channels**: When a child workflow uses `EngineOnly` (no custom error) and the parent returns `dbos::Result<R, MyError>`, call `.lift()` on `child.start(...)` and `handle.result()`. When the child has its own error type `ChildErr`, convert with `.map_error(MyError::from)`.
- **Cancellation isolation**: If an awaited child is cancelled, `handle.result()` inside the parent returns `Err(Error::AwaitedWorkflowCancelled { workflow_id })` (checkpointed as that step's result) rather than `Error::WorkflowCancelled`, so catching or propagating it will not trick the engine into thinking the parent itself was cancelled.
- **`retrieve_workflow` missing-ID behavior**: Neither `DBOS::retrieve_workflow` nor `Client::retrieve_workflow` queries PostgreSQL on construction. Calling `handle.status().await` immediately returns `Err(Error::WorkflowNotFound)` if no row exists, whereas `handle.result().await` polls waiting for the row to appear (in case an upstream enqueue has not committed yet). Call `status()` first or wrap `result()` in `tokio::time::timeout` outside workflows when the ID may be mistyped.

Reference: [dbos::WorkflowRef::start](https://docs.rs/dbos/latest/dbos/struct.WorkflowRef.html#method.start), [dbos::WorkflowHandle](https://docs.rs/dbos/latest/dbos/struct.WorkflowHandle.html), [dbos::DBOS::retrieve_workflow](https://docs.rs/dbos/latest/dbos/struct.DBOS.html#method.retrieve_workflow)
