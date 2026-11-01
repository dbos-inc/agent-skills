---
title: Respect Ambient Context and Step Placement Constraints
impact: CRITICAL
impactDescription: Prevents runtime InsideStep, NotInWorkflow, and StepBuiltElsewhere errors and avoids burning step IDs
tags: workflow, step, constraints, PendingStep, context, tokio-spawn
---

## Respect Ambient Context and Step Placement Constraints

DBOS Rust tracks workflow identity and the sequential `step_id` counter in a `tokio::task_local!` context (`Ctx`). Durable primitives check where they are called, claim their `step_id` eagerly **when the future is constructed** (not when first polled), and refuse to be awaited in a different workflow or step from the one that built them.

**Incorrect (calling workflow-only APIs inside a step, spawning tasks, and dropping unawaited steps):**

```rust
use dbos::WorkflowRef;

async fn broken_orchestrator(child: WorkflowRef<u32, u32>, x: u32) -> dbos::Result<u32> {
    // Bug 1: Claiming a PendingStep and dropping it without `.await` still increments `step_id`!
    let _unused = dbos::step("abandoned", || async { Ok::<_, dbos::Error>(1_u32) });

    // Bug 2: `tokio::spawn` does NOT inherit `tokio::task_local!` workflow context!
    let spawned = tokio::spawn(async move {
        // Runs outside any workflow context: `set_event` fails with `Error::NotInWorkflow`!
        dbos::set_event::<_, dbos::EngineOnly>("progress", &x).await
    });
    let _ = spawned.await;

    // Bug 3: Starting a child workflow, setting events, or calling `recv` inside `dbos::step`
    // fails with `Error::InsideStep`!
    dbos::step("bad_step", || {
        let child = child.clone();
        async move { child.run(x).await }
    })
    .await
}
```

**Correct (keeping durable orchestration in the workflow body and leaf I/O in steps):**

```rust
use dbos::WorkflowRef;

async fn valid_orchestrator(child: WorkflowRef<u32, u32>, x: u32) -> dbos::Result<u32> {
    // 1. Run leaf work in a step and await it immediately
    let prepared = dbos::step("prepare_input", || async move { Ok(x + 1) }).await?;

    // 2. Publish events, receive messages, and start child workflows in the workflow body
    dbos::set_event("progress", &prepared).await?;
    let child_result = child.run(prepared).await?;

    // 3. Concurrent steps in the same task via `tokio::join!` keep deterministic step IDs
    let (left, right) = tokio::join!(
        dbos::step("left_branch", || async move { Ok(child_result * 2) }),
        dbos::step("right_branch", || async move { Ok(child_result + 10) }),
    );

    Ok(left? + right?)
}
```

### Placement Matrix Across Contexts

- **Refused inside `dbos::step` (`Error::InsideStep`)**: `WorkflowRef::{run, run_with, start, start_with}`, `Debouncer::{debounce, debounce_with}`, `dbos::set_event` / `set_event_with`, `dbos::recv`, `dbos::patch`, and `dbos::deprecate_patch`.
- **Refused outside any workflow (`Error::NotInWorkflow`)**: Free functions `dbos::set_event`, `dbos::get_event`, `dbos::recv`, `dbos::send` / `send_bulk`, `dbos::write_stream` / `close_stream` / `read_stream`, `dbos::select_workflow` / `join_workflows`, `dbos::patch` / `deprecate_patch`. (Use the corresponding methods on `DBOS` or `Client` when outside a workflow.)
- **Allowed in both workflow body and step body**: `dbos::write_stream` and `dbos::close_stream` (checkpointed as steps in the workflow body; appended plainly inside a step body), `dbos::send_with` (plain inside a step; pass `SendMessage::idempotency_key` so retries deduplicate), `dbos::get_event`, `dbos::step` (plain leaf call when nested), and `dbos::sleep` (plain `tokio::time::sleep` when nested).
- **Never move a `PendingStep` or `WorkflowHandle::result()` across contexts**: Constructing `let fut = handle.result();` in the workflow body and awaiting `fut` inside `dbos::step` or `tokio::spawn` fails with `Error::StepBuiltElsewhere`. Similarly, awaiting a handle created on another `DBOS` instance fails with `Error::WrongInstance`.

Reference: [dbos::PendingStep](https://docs.rs/dbos/latest/dbos/struct.PendingStep.html), [dbos::Error](https://docs.rs/dbos/latest/dbos/enum.Error.html)
