---
title: Enqueue Workflows from External Applications
impact: HIGH
impactDescription: Type mismatches with the remote workflow surface only at decode time
tags: client, enqueue, external
---

## Enqueue Workflows from External Applications

`client.enqueue(name, queue, input)` and `enqueue_with(name, input, EnqueueOptions { .. })` enqueue a workflow by name; whichever executor has the code dequeues it. The handle's type parameters state what the workflow returns; nothing checks them until `result()` decodes. `EnqueueOptions` carries the `QueueRequest` fields plus `workflow_id`, `app_name`, `app_version`, `timeout` and `attributes`.

**Incorrect (the wrong result type):**

```rust
async fn enqueue_bad(client: &Client) -> dbos::Result<()> {
    // WRONG: process_order returns a String, but nothing checks the type
    // parameter: the mismatch only surfaces as a decoding error from result().
    let handle: WorkflowHandle<u64> = client
        .enqueue("process_order", "orders", "order-123")
        .await?;
    let total = handle.result().await?;
    Ok(())
}
```

**Correct (an ID and the right type):**

```rust
async fn enqueue_order(client: &Client) -> dbos::Result<String> {
    let options = EnqueueOptions {
        workflow_id: Some("order-123"),
        ..EnqueueOptions::new("orders")
    };
    // The type parameter states what the workflow returns; nothing checks it
    // until the result is decoded.
    let handle: WorkflowHandle<String> = client
        .enqueue_with("process_order", "order-123", options)
        .await?;
    handle.result().await
}
```

A client also has `send`, `get_event`, `read_stream`, `list_workflows`, `cancel`/`resume`/`fork`/`rewind`/`delete`, queue and schedule management.
