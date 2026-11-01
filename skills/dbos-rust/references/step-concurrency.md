---
title: Race Durable Calls Deterministically with select_step!
impact: HIGH
impactDescription: Records the winning branch index so crash recovery replays the exact branch that won
tags: step, concurrency, select_step, race, lift, map_error
---

## Race Durable Calls Deterministically with select_step!

Never use `tokio::select!` to race durable operations inside a workflow. Even though `PendingStep` futures claim their `step_id` deterministically when constructed, `tokio::select!` never records which branch won, so a replay after a crash can pick a different branch. Use `dbos::select_step!` to race heterogeneous `PendingStep` calls (steps, `sleep`, `get_event`, `set_event`, `handle.result()`, or `select_workflow`) with a checkpointed winner index.

**Incorrect (tokio::select! over steps and trying to race WorkflowRef::start directly):**

```rust
use std::time::Duration;

async fn charge_with_deadline(cents: u32) -> dbos::Result<String> {
    // Broken on recovery: nothing records whether `charge` or `sleep` won!
    tokio::select! {
        res = dbos::step("charge", || async move { Ok(format!("paid {cents}")) }) => res,
        res = dbos::sleep(Duration::from_secs(30)) => {
            res?;
            Ok("timed out".to_owned())
        }
    }
}
```

**Correct (dbos::select_step! with unified error types and pre-started child handles):**

```rust
use std::time::Duration;
use dbos::{WorkflowRef, select_step};
use serde::{Deserialize, Serialize};

#[derive(Debug, thiserror::Error, Serialize, Deserialize)]
#[error("billing error: {0}")]
struct BillingError(String);

async fn charge_or_cancel(
    fraud_check: WorkflowRef<u32, bool>,
    cents: u32,
) -> dbos::Result<String, BillingError> {
    // WorkflowRef::start / run cannot be placed inside select_step! (refused by type):
    // start the child outside the race, then race `handle.result()`
    let fraud_handle = fraud_check.start(cents).lift().await?;
    let fraud_id = fraud_handle.workflow_id().to_owned();

    select_step! {
        flagged = fraud_handle.result().lift() => {
            if flagged? {
                "rejected by fraud check".to_owned()
            } else {
                format!("approved {cents}")
            }
        }
        // Sleep takes `BillingError` via type inference; if this arm wins, fraud_handle's
        // child workflow keeps running unless explicitly cancelled
        expired = dbos::sleep(Duration::from_secs(10)) => {
            expired?;
            let _ = fraud_id;
            "fraud check timed out".to_owned()
        }
    }
}
```

### Rules and Edge Cases of `select_step!`

- **Requires the default `macros` Cargo feature, and `dbos` must not be renamed**: `select_step!` expands to paths rooted at `::dbos`, so renaming the dependency in `Cargo.toml` (`my_dbos = { package = "dbos" }`) fails to resolve.
- **When every branch is a workflow handle, prefer `dbos::select_workflow!`**: `select_workflow!` polls all handles in a single SQL query per interval, whereas N `handle.result()` arms in `select_step!` issue N separate polling queries (see [workflow-concurrency.md](workflow-concurrency.md)).
- **Compile-time refusals for replay safety**: `select_step!` requires at least 2 arms and rejects `if` guards, `else`, `complete`, refutable arm patterns, and `biased;` with targeted compile errors because any conditional arm set would shift branch indices on replay.
- **Unify error types on the call, before `=>`**: All branches must share one `DurableError` type `E`. Convert `EngineOnly` calls with `.lift()` and convert foreign application error types with `.map_error(|e| MyError::from(e))` directly on the branch expression.
- **What happens to losing branches**: Losing `step` futures are dropped without checkpointing and their `dbos::cancellation_token()` fires. Losing `sleep` calls leave behind the wake-time checkpoint written before waiting. Losing `handle.result()` calls drop only the wait while the child workflow continues running in the background; bound the child with `StartOptions::timeout` or cancel it explicitly if the child should stop when it loses.

Reference: [dbos::select_step](https://docs.rs/dbos/latest/dbos/macro.select_step.html), [dbos::PendingStep::lift](https://docs.rs/dbos/latest/dbos/struct.PendingStep.html#method.lift), [dbos::PendingStep::map_error](https://docs.rs/dbos/latest/dbos/struct.PendingStep.html#method.map_error)
