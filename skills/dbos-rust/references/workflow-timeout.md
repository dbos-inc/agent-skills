---
title: Bound Workflow Execution with Durable Timeouts
impact: HIGH
impactDescription: Enforces deadlines on the database clock across crashes, queues, and child workflows
tags: workflow, timeout, deadline, cancellation, Timeout
---

## Bound Workflow Execution with Durable Timeouts

Set workflow deadlines with `Timeout` on `RunOptions`, `StartOptions`, or `EnqueueOptions` rather than wrapping workflow calls in `tokio::time::timeout`. A `tokio::time::timeout` only drops the caller's waiting future in the current process while leaving the workflow running in PostgreSQL; a durable `Timeout` stamps a database-clock deadline on the workflow row and cancels the workflow across restarts and across the worker fleet.

**Incorrect (caller-side tokio::time::timeout leaves the workflow running):**

```rust
use std::time::Duration;
use dbos::WorkflowRef;

async fn trigger_report(report_wf: &WorkflowRef<String, String>) -> Result<String, String> {
    // Drops only the local PendingRun future; if already started or recovered,
    // the workflow continues running indefinitely in the background!
    tokio::time::timeout(
        Duration::from_secs(60),
        report_wf.run("q4-summary".to_owned()),
    )
    .await
    .map_err(|_| "timed out locally".to_owned())?
    .map_err(|e| e.to_string())
}
```

**Correct (durable Timeout on RunOptions / StartOptions, plus detaching background children):**

```rust
use std::time::Duration;
use dbos::{RunOptions, StartOptions, Timeout, WorkflowRef};

async fn generate_report(
    audit_wf: WorkflowRef<String, ()>,
    report_name: String,
) -> dbos::Result<String> {
    // Detach background audit child with Timeout::None so it is not cancelled
    // if the parent's 60-second deadline expires later
    let _audit_handle = audit_wf
        .start_with(
            report_name.clone(),
            StartOptions {
                timeout: Timeout::None,
                ..StartOptions::default()
            },
        )
        .await?;

    dbos::step("render_pdf", || async { Ok(format!("{report_name}.pdf")) }).await
}

async fn trigger_report(report_wf: &WorkflowRef<String, String>) -> dbos::Result<String> {
    report_wf
        .run_with(
            "q4-summary".to_owned(),
            RunOptions {
                timeout: Timeout::Explicit(Duration::from_secs(60)), // or Duration::from_secs(60).into()
                ..RunOptions::default()
            },
        )
        .await
}
```

### How `Timeout` Variants and Deadlines Work

- `Timeout::Inherit` *(default)*: Root workflows run with no deadline. Child workflows started inside a parent inherit the parent's exact wall-clock deadline (even when enqueued onto a queue).
- `Timeout::Explicit(Duration)` *(also `Duration::into()`)*: Sets a wall-clock budget computed against the PostgreSQL clock. For direct starts, the clock starts at row insertion; for queued workflows, the deadline is stamped **when a worker claims the workflow from the queue**, so queue wait time does not consume execution budget. Inside a parent workflow, `Explicit` replaces any inherited parent deadline. `Duration::ZERO` is rejected with `Error::InvalidArgument`.
- `Timeout::None`: Runs with no deadline, clearing any deadline that would have been inherited from a parent workflow.

### Cancellation Semantics on Expiry

- Every launched `DBOS` executor runs a background timeout sweep against the database clock, marking expired unsettled workflows (`PENDING`, `ENQUEUED`, or `DELAYED`) as `CANCELLED` even if no process is actively polling them.
- In-flight workflows stop at their next step boundary (or mid-step if the step sets `StepOptions { preemptible: true, .. }`, which also fires `dbos::cancellation_token()`).
- Outside a workflow, awaiting a timed-out handle returns `Err(Error::WorkflowCancelled { workflow_id })`. Inside a parent workflow, awaiting a timed-out child returns `Err(Error::AwaitedWorkflowCancelled { workflow_id })`, recorded as that step's result so the parent can handle or compensate for the child's timeout without cancelling itself.

Reference: [dbos::Timeout](https://docs.rs/dbos/latest/dbos/enum.Timeout.html), [dbos::RunOptions](https://docs.rs/dbos/latest/dbos/struct.RunOptions.html)
