---
title: Use Messages for Workflow Notifications
impact: MEDIUM
impactDescription: Polling for external signals wastes queries and is not durable
tags: communication, messages, send, recv, notifications
---

## Use Messages for Workflow Notifications

Send a message to a workflow with `dbos.send(id, &msg)` (or `send_with` and a `SendMessage` with a `topic`); a workflow waits for one with `dbos::recv(topic, timeout)`, which returns `None` on timeout. Messages are stored, so one sent before the workflow reaches `recv` waits for it. A send from a workflow is exactly-once; from a step it repeats if the step retries, unless it carries `SendMessage::idempotency_key`.

**Incorrect (polling a flag):**

```rust
async fn wait_bad(_: ()) -> dbos::Result<Option<String>> {
    // WRONG: polling a database flag in a loop wastes queries and isn't durable.
    loop {
        let paid = step("check_paid", || async { Ok(false) }).await?;
        if paid {
            return Ok(Some("paid".to_owned()));
        }
        tokio::time::sleep(Duration::from_secs(5)).await;
    }
}
```

**Correct (recv with a timeout, send from the webhook):**

```rust
const PAYMENT_STATUS: &str = "payment_status";

async fn create_order() -> dbos::Result<()> {
    Ok(())
}
async fn fulfill_order() -> dbos::Result<()> {
    Ok(())
}
async fn cancel_order() -> dbos::Result<()> {
    Ok(())
}

async fn checkout_workflow(_: ()) -> dbos::Result<()> {
    dbos::step("create_order", create_order).await?;
    // Wait up to an hour for the payment service to report.
    let status: Option<String> =
        dbos::recv(Some(PAYMENT_STATUS), Duration::from_secs(60 * 60)).await?;
    match status.as_deref() {
        Some("paid") => dbos::step("fulfill_order", fulfill_order).await?,
        // Cancelled, or no answer before the timeout.
        _ => dbos::step("cancel_order", cancel_order).await?,
    }
    Ok(())
}

// The payment service's webhook handler.
async fn payment_webhook(dbos: &DBOS, workflow_id: &str, status: &str) -> dbos::Result<()> {
    let message = SendMessage {
        topic: Some(PAYMENT_STATUS),
        ..SendMessage::new(workflow_id, &status)
    };
    dbos.send_with(message, SendOptions::default()).await
}
```

`recv` is workflow-body only. `send_bulk` sends many messages in one transaction.
