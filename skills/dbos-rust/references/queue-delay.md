---
title: Defer Queued Workflows with Enqueue::delay and set_workflow_delay
impact: LOW-MEDIUM
impactDescription: Holds workflows in DELAYED status on the database clock without occupying worker tasks
tags: queue, delay, WorkflowDelay, set_workflow_delay, scheduled
---

## Defer Queued Workflows with Enqueue::delay and set_workflow_delay

When an entire workflow should wait before starting, enqueue it with `Enqueue { delay: Some(duration), .. }` rather than starting it immediately and calling `dbos::sleep` on line 1. A delayed enqueue inserts the workflow in `WorkflowStatus::Delayed` with zero in-memory task overhead; the background supervisor promotes it to `WorkflowStatus::Enqueued` once `delay_until` passes on the PostgreSQL clock. While a workflow is still `DELAYED`, you can reschedule or release it early with `DBOS::set_workflow_delay`.

**Incorrect (holding an active worker slot just to sleep before any step runs):**

```rust
use std::time::Duration;
use dbos::{Enqueue, StartOptions, WorkflowRef};

async fn send_abandoned_cart_email(cart_id: String) -> dbos::Result<()> {
    // Holds a `PENDING` worker concurrency slot for 2 hours before doing any work!
    dbos::sleep(Duration::from_secs(2 * 3600)).await?;
    dbos::step("send_email", || async move {
        let _ = cart_id;
        Ok(())
    })
    .await
}

async fn schedule_reminder(
    wf: &WorkflowRef<String, ()>,
    cart_id: String,
) -> dbos::Result<()> {
    wf.start_with(
        cart_id,
        StartOptions {
            queue: Some(Enqueue::new("emails")),
            ..StartOptions::default()
        },
    )
    .await?;
    Ok(())
}
```

**Correct (enqueuing in DELAYED status and adjusting delay_until dynamically):**

```rust
use std::time::Duration;
use dbos::{DBOS, Enqueue, StartOptions, WorkflowDelay, WorkflowRef};

async fn send_abandoned_cart_email(cart_id: String) -> dbos::Result<()> {
    dbos::step("send_email", || {
        let cart_id = cart_id.clone();
        async move {
            let _ = cart_id;
            Ok(())
        }
    })
    .await
}

async fn schedule_and_reschedule(
    dbos: &DBOS,
    wf: &WorkflowRef<String, ()>,
    cart_id: String,
) -> dbos::Result<String> {
    // Inserted as `DELAYED`; consumes zero `concurrency` / `worker_concurrency` slots while waiting
    let handle = wf
        .start_with(
            cart_id,
            StartOptions {
                queue: Some(Enqueue {
                    delay: Some(Duration::from_secs(2 * 3600)),
                    ..Enqueue::new("emails")
                }),
                ..StartOptions::default()
            },
        )
        .await?;
    let wf_id = handle.workflow_id().to_owned();

    // Push out by 30 minutes from now on DB clock (or pass `Duration::ZERO` to release immediately!)
    dbos.set_workflow_delay(&wf_id, WorkflowDelay::For(Duration::from_secs(30 * 60)))
        .await?;

    Ok(wf_id)
}
```

### Delayed Workflow Details

- **Database clock authority**: Both `Enqueue::delay` and `WorkflowDelay::For(Duration)` compute `delay_until` on the PostgreSQL clock (`WorkflowDelay::Until(Timestamp)` sets an explicit `dbos::sysdb::types::Timestamp`).
- **`Enqueue::delay` vs `set_workflow_delay` zero handling**: At submission, `Enqueue { delay: Some(Duration::ZERO), .. }` is rejected (`delay` must be `> Duration::ZERO`; omit `delay` to run immediately). On an already `DELAYED` workflow, `set_workflow_delay(id, WorkflowDelay::For(Duration::ZERO))` (or a past `Until` timestamp) is valid and promotes the row to ready on the next sweep.
- **No-op once promoted**: `set_workflow_delay` updates the row **only** while its status is `DELAYED`; if the workflow has already transitioned to `ENQUEUED`, `PENDING`, or terminal, the call succeeds as a safe no-op without moving it backward (`NonExistentWorkflow` is returned only when the `workflow_id` does not exist).

Reference: [dbos::Enqueue::delay](https://docs.rs/dbos/latest/dbos/struct.Enqueue.html#structfield.delay), [dbos::DBOS::set_workflow_delay](https://docs.rs/dbos/latest/dbos/struct.DBOS.html#method.set_workflow_delay), [dbos::WorkflowDelay](https://docs.rs/dbos/latest/dbos/sysdb/types/enum.WorkflowDelay.html)
