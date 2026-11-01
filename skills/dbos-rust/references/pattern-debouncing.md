---
title: Coalesce Rapid Bursts into One Run with Debouncer
impact: MEDIUM
impactDescription: Collapses bursty per-key updates into a single workflow run with the latest input and an optional max-wait cap
tags: pattern, debouncing, Debouncer, DebouncerClient, DebounceOptions, debounce_timeout
---

## Coalesce Rapid Bursts into One Run with Debouncer

Use `dbos::Debouncer` (from a `DBOS` process) or `dbos::DebouncerClient` (from an external `Client`) when rapid updates for the same key (autosave edits, search re-indexing, cart syncs) should collapse into **one** workflow execution carrying the **latest** input. The first call enqueues a `DELAYED` workflow holding `<workflow>-<key>`; subsequent calls before it starts push its release out by `period` and overwrite its input, capped at `debounce_timeout` from the first call.

**Incorrect (enqueuing every keystroke separately or passing Duration::ZERO):**

```rust
use std::time::Duration;
use dbos::{Debouncer, WorkflowRef};

async fn broken_debounce(
    index_doc: &WorkflowRef<(String, String), ()>,
    doc_id: &str,
    content: String,
) -> dbos::Result<()> {
    let debouncer = Debouncer::new(index_doc);

    // Fails with `Error::InvalidArgument`: `period` (and `debounce_timeout` when `Some`)
    // must be strictly greater than `Duration::ZERO`!
    debouncer
        .debounce(doc_id, Duration::ZERO, (doc_id.to_owned(), content))
        .await?;
    Ok(())
}
```

**Correct (reusable Debouncer with quiet period and maximum debounce_timeout cap):**

```rust
use std::time::Duration;
use dbos::{Client, DebounceOptions, Debouncer, DebouncerClient, WorkflowHandle, WorkflowRef};

// Inside the DBOS application: construct once (or cheaply per call) over `&WorkflowRef`
async fn coalesce_document_edits(
    index_doc: &WorkflowRef<(String, String), usize>,
    doc_id: &str,
    latest_markdown: String,
) -> dbos::Result<WorkflowHandle<usize>> {
    let debouncer = Debouncer {
        queue: Some("search-indexing"),
        // Even under continuous edits every second, guarantee indexing runs within 30s!
        debounce_timeout: Some(Duration::from_secs(30)),
        timeout: Some(Duration::from_secs(120)),
        ..Debouncer::new(index_doc)
    };

    // Waits for 2s of quiet after the most recent edit (or until the 30s cap), then runs
    // ONCE with the latest `(doc_id, latest_markdown)` tuple
    debouncer
        .debounce_with(
            doc_id,
            Duration::from_secs(2),
            (doc_id.to_owned(), latest_markdown),
            DebounceOptions::default(),
        )
        .await
}

// From an external `dbos::Client` process: use `DebouncerClient`
async fn debounce_from_gateway(
    client: &Client,
    user_id: &str,
    cart_json: String,
) -> dbos::Result<WorkflowHandle<()>> {
    let debouncer = DebouncerClient {
        debounce_timeout: Some(Duration::from_secs(60)),
        ..DebouncerClient::new(client, "sync_cart", "cart-sync")
    };
    debouncer
        .debounce(user_id, Duration::from_secs(5), cart_json)
        .await
}
```

### Debouncing Rules and Edge Cases

- **Latest input wins, same handle returned**: Every call that coalesces onto the waiting `DELAYED` workflow updates its stored input to the newest value and returns a `WorkflowHandle` pointing at that shared `workflow_id`. Once a worker claims the workflow from the queue, the key is freed so the next edit starts a fresh debounce window.
- **No priority or partition key**: Debounced workflows do not accept `priority` or `partition_key`; always target an **unpartitioned** queue (`Debouncer::queue: None` uses `_dbos_internal_queue`, or pass `Some("unpartitioned-queue")`).
- **`DebounceOptions::workflow_id` applies only to fresh inserts**: When a call coalesces onto an already-waiting workflow, the existing workflow keeps its ID and `DebounceOptions::workflow_id` is ignored.
- **Calling `Debouncer` inside a workflow**: Allowed in a workflow body (checkpointed for replay; refused inside `dbos::step` with `Error::InsideStep`). Note that a fresh enqueue inside a parent workflow records `parent_workflow_id`, so cancelling the parent with `Children::Include` also cancels that shared debounced child.

Reference: [dbos::Debouncer](https://docs.rs/dbos/latest/dbos/struct.Debouncer.html), [dbos::DebouncerClient](https://docs.rs/dbos/latest/dbos/struct.DebouncerClient.html), [dbos::DebounceOptions](https://docs.rs/dbos/latest/dbos/struct.DebounceOptions.html)
