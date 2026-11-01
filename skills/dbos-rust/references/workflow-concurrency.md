---
title: Coordinate Multiple Workflows with join_workflows and select_workflow
impact: HIGH
impactDescription: Polls fan-out workflow sets in one SQL query per interval and checkpoints race winners deterministically
tags: workflow, concurrency, fan-out, join_workflows, select_workflow, race
---

## Coordinate Multiple Workflows with join_workflows and select_workflow

When waiting on multiple child or queued workflows, use `dbos::join_workflows!` / `dbos::join_workflows` to wait for all of them or `dbos::select_workflow!` / `dbos::select_workflow` to wait for the first one to settle. Both issue a **single SQL query per poll interval** across the entire set, and `select_workflow` checkpoints the winning `workflow_id` so replays take the exact same branch.

**Incorrect (tokio::select! over workflow handles or comma instead of semicolon outside a workflow):**

```rust
use dbos::{DBOS, WorkflowHandle};

async fn broken_race(
    a: WorkflowHandle<u32>,
    b: WorkflowHandle<String>,
) -> dbos::Result<String> {
    // Non-deterministic: tokio::select! does not record which child settled first!
    tokio::select! {
        res = a.result() => Ok(res?.to_string()),
        res = b.result() => res,
    }
}

async fn broken_outside_join(
    dbos: &DBOS,
    a: WorkflowHandle<u32>,
    b: WorkflowHandle<u32>,
) -> dbos::Result<u32> {
    // Compile error: `dbos` separated by a comma is parsed as a third WorkflowHandle!
    // Outside a workflow, separate the `DBOS` or `Client` instance with a semicolon (`;`)
    let (x, y) = dbos::join_workflows!(dbos, a, b)?;
    Ok(x + y)
}
```

**Correct (using join_workflows!, select_workflow!, and dynamic ID drain loops):**

```rust
use dbos::{DBOS, WorkflowHandle, WorkflowRef};

// 1. Fixed-arity heterogeneous fan-out and race inside a workflow (no `.await` after the macro!)
async fn race_or_join_children(
    price_wf: WorkflowRef<String, u64>,
    eta_wf: WorkflowRef<String, String>,
    sku: String,
) -> dbos::Result<String> {
    // Bind handles to local variables first: both macros require `ident` bindings, not expressions
    let price_h = price_wf.start(sku.clone()).await?;
    let eta_h = eta_wf.start(sku.clone()).await?;
    let (price, eta) = dbos::join_workflows!(price_h, eta_h)?;

    let fast_quote = price_wf.start(format!("{sku}-fast")).await?;
    let backup_quote = eta_wf.start(format!("{sku}-backup")).await?;

    // Checkpoints `DBOS.select_workflow` with the winning ID, then awaits only the winner's result
    let winner_summary = dbos::select_workflow! {
        cents = fast_quote => format!("quote {} cents", cents?),
        note = backup_quote => format!("backup {}", note?),
    }?;

    Ok(format!("{price} ({eta}) / {winner_summary}"))
}

// 2. Dynamic slice drain loop inside a workflow (completion order, 1 SQL query per pass)
async fn drain_batch(worker: WorkflowRef<u32, u32>, items: Vec<u32>) -> dbos::Result<Vec<u32>> {
    let mut handles = Vec::with_capacity(items.len());
    for item in items {
        handles.push(worker.start(item).await?);
    }

    let mut completed = Vec::with_capacity(handles.len());
    while !handles.is_empty() {
        let winner_id = {
            let ids: Vec<&str> = handles.iter().map(WorkflowHandle::workflow_id).collect();
            dbos::select_workflow(&ids).await?
        };
        let idx = handles
            .iter()
            .position(|h| h.workflow_id() == winner_id)
            .expect("winner came from handles");
        completed.push(handles.swap_remove(idx).result().await?);
    }
    Ok(completed)
}

// 3. Outside a workflow (HTTP handler, CLI, or Client): pass `dbos;` before a semicolon
async fn join_from_handler(
    dbos: &DBOS,
    first: WorkflowHandle<u32>,
    second: WorkflowHandle<u32>,
) -> dbos::Result<u32> {
    let (a, b) = dbos::join_workflows!(dbos; first, second)?;
    Ok(a + b)
}
```

### Key Rules for `join_workflows` and `select_workflow`

- **Macros expand `.await` internally**: Both `join_workflows!(...)` and `select_workflow! { ... }` evaluate directly to `dbos::Result<T, E>` (do not append `.await`). In `select_workflow!`, each arm binds the winner's `dbos::Result<R, E>` and returns `T` (which the macro wraps in `Ok(...)`).
- **Pass handle variables (`ident`), not expressions**: Both macros read `handle.workflow_id()` and then consume `handle.result()`, so passing `child.start(x).await?` inline is rejected at compile time to prevent starting the child twice.
- **Settled means terminal, not just `SUCCESS`**: A workflow finishes waiting as soon as its status leaves `PENDING`, `ENQUEUED`, and `DELAYED` (so `ERROR`, `CANCELLED`, and `MAX_RECOVERY_ATTEMPTS_EXCEEDED` also settle the wait).
- **Losing workflows keep running**: Unlike `select_step!` dropping an in-memory step future, losing handles in `select_workflow!` represent durable workflows that continue executing unless explicitly cancelled or bounded with `StartOptions::timeout`.

Reference: [dbos::join_workflows](https://docs.rs/dbos/latest/dbos/macro.join_workflows.html), [dbos::select_workflow](https://docs.rs/dbos/latest/dbos/macro.select_workflow.html)
