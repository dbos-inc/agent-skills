---
title: Publish and Read Workflow State with set_event and get_event
impact: MEDIUM
impactDescription: Exposes latest key-value progress from running or completed workflows without custom polling tables
tags: comm, events, set_event, get_event, progress, state
---

## Publish and Read Workflow State with set_event and get_event

Use `dbos::set_event(key, &value).await?` inside a workflow body to publish or overwrite a durable key-value entry on that workflow, and read it via `dbos::get_event` (inside a workflow), `dbos.get_event` (from an HTTP handler or service), or `client.get_event` (from an external client). Events hold **only the latest value per key** (overwriting an existing key updates it in place) and remain readable even after the workflow finishes.

**Incorrect (calling set_event inside a step or treating get_event timeout as an Err):**

```rust
use std::time::Duration;
use dbos::DBOS;

async fn broken_event_workflow(order_id: String) -> dbos::Result<()> {
    dbos::step("charge", || {
        let order_id = order_id.clone();
        async move {
            // Fails with `Error::InsideStep`! `set_event` must stand in the workflow body
            dbos::set_event("payment_id", &format!("pay_{order_id}")).await?;
            Ok(())
        }
    })
    .await
}

async fn poll_wrong(dbos: &DBOS, wf_id: &str) -> dbos::Result<Option<String>> {
    // Blocks up to 30 seconds instead of returning immediately on a status-bar poll!
    dbos.get_event(wf_id, "payment_id", Duration::from_secs(30)).await
}
```

**Correct (publishing in workflow body, blocking for handoff, and polling with Duration::ZERO):**

```rust
use std::time::Duration;
use dbos::{DBOS, StartOptions, WorkflowRef};

const PAYMENT_ID_KEY: &str = "payment_id";
const PROGRESS_KEY: &str = "progress_pct";

async fn checkout_workflow(sku: String) -> dbos::Result<String> {
    let payment_id = dbos::step("create_payment_intent", || {
        let sku = sku.clone();
        async move { Ok(format!("pi_{sku}")) }
    })
    .await?;

    // Checkpointed as step `DBOS.set_event`; skipped on replay
    dbos::set_event(PAYMENT_ID_KEY, &payment_id).await?;
    dbos::set_event(PROGRESS_KEY, &50_u32).await?;

    let receipt = dbos::step("fulfill_shipment", || {
        let payment_id = payment_id.clone();
        async move { Ok(format!("shipped:{payment_id}")) }
    })
    .await?;

    // Overwrites `PROGRESS_KEY` with the latest value
    dbos::set_event(PROGRESS_KEY, &100_u32).await?;
    Ok(receipt)
}

// Blocking handoff: start workflow in background and wait only until the first event lands
async fn start_checkout_handler(
    dbos: &DBOS,
    checkout: &WorkflowRef<String, String>,
    idempotency_key: &str,
    sku: String,
) -> dbos::Result<Option<String>> {
    checkout
        .start_with(
            sku,
            StartOptions {
                workflow_id: Some(idempotency_key),
                ..StartOptions::default()
            },
        )
        .await?;

    // Returns `Ok(Some(id))` as soon as `set_event` commits (via LISTEN/NOTIFY),
    // or `Ok(None)` if 15 seconds elapse without publishing
    dbos.get_event(idempotency_key, PAYMENT_ID_KEY, Duration::from_secs(15))
        .await
}

// Non-blocking poll: `Duration::ZERO` checks PostgreSQL once and returns immediately
async fn read_current_progress(dbos: &DBOS, workflow_id: &str) -> dbos::Result<u32> {
    let pct: Option<u32> = dbos
        .get_event(workflow_id, PROGRESS_KEY, Duration::ZERO)
        .await?;
    Ok(pct.unwrap_or(0))
}
```

### Event Semantics Across Surfaces

- **Timeout returns `Ok(None)`, not `Err`**: Absence of a key when `timeout` expires is `Ok(None)`. Pass `Duration::ZERO` for an immediate single-read poll with no wait.
- **Three `get_event` surfaces**:
  - Inside a workflow body: `dbos::get_event(workflow_id, key, timeout).await?` (checkpointed as two steps so replays return the exact `Option<T>` observed on the first run without waiting again; avoids capturing `DBOS` in the workflow closure).
  - Outside a workflow on `DBOS`: `dbos.get_event::<T>(workflow_id, key, timeout).await?`.
  - From an external client: `client.get_event::<T>(workflow_id, key, timeout).await?` (note: set `ClientConfig::use_listen_notify = true` only when executors also run on PostgreSQL with `use_listen_notify: true`).
- **Cross-language events**: Use `dbos::set_event_with(key, &val, SetEventOptions { serialization: SerializationFormat::Portable })` when a non-portable Rust workflow publishes an event read by Python, TypeScript, Go, or Java.

Reference: [dbos::set_event](https://docs.rs/dbos/latest/dbos/fn.set_event.html), [dbos::get_event](https://docs.rs/dbos/latest/dbos/fn.get_event.html), [dbos::DBOS::get_event](https://docs.rs/dbos/latest/dbos/struct.DBOS.html#method.get_event)
