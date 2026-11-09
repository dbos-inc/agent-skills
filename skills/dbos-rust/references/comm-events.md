---
title: Use Events for Workflow Status Publishing
impact: MEDIUM
impactDescription: Callers cannot see a workflow's progress without polling its result
tags: communication, events, set_event, get_event
---

## Use Events for Workflow Status Publishing

A workflow publishes key-value events with `dbos::set_event(key, &value)` from its body (not from a step: `Error::InsideStep`). Read them with `dbos.get_event(workflow_id, key, timeout)` outside a workflow, or `dbos::get_event` inside one (recorded); both wait for the key and return `None` on timeout. The latest value of a key wins.

**Incorrect (set_event inside a step):**

```rust
async fn publish_bad(_: ()) -> dbos::Result<()> {
    // WRONG: set_event is refused inside a step (Error::InsideStep).
    step("publish", || async {
        dbos::set_event("status", &"processing").await?;
        Ok(())
    })
    .await
}
```

**Correct (publish from the workflow, read from the handler):**

```rust
const PAYMENT_URL: &str = "payment_url";

async fn create_payment_session() -> dbos::Result<String> {
    Ok("https://pay.example.com/session/123".to_owned())
}

async fn payment_workflow(_: ()) -> dbos::Result<()> {
    let url = dbos::step("create_payment_session", create_payment_session).await?;
    // Publish the URL for the web handler to read.
    dbos::set_event(PAYMENT_URL, &url).await?;
    // ... wait for the payment, as in the messages example ...
    Ok(())
}

// The web handler starts the workflow, then waits for it to publish the URL.
async fn start_checkout(
    dbos: &DBOS,
    payment_workflow: &WorkflowRef<(), ()>,
) -> dbos::Result<Option<String>> {
    let handle = payment_workflow.start(()).await?;
    let url: Option<String> = dbos
        .get_event(handle.workflow_id(), PAYMENT_URL, Duration::from_secs(60))
        .await?;
    Ok(url)
}
```
