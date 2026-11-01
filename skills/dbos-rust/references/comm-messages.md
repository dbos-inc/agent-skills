---
title: Signal Waiting Workflows with Durable Messages (send and recv)
impact: MEDIUM
impactDescription: Delivers FIFO topic-scoped signals to workflows with atomic consumption and webhook idempotency
tags: comm, messages, send, send_with, send_bulk, recv, webhook, approval
---

## Signal Waiting Workflows with Durable Messages (send and recv)

Use `dbos::recv(topic, timeout).await?` inside a workflow body to pause durably for incoming signals (webhook callbacks, human approvals, external events), and deliver messages via `dbos::send` / `send_with` / `send_bulk` (inside a workflow) or the matching methods on `DBOS` and `Client` (outside a workflow). Unlike events (which overwrite one key), messages queue in FIFO order per topic and each `recv` atomically consumes one message in the same transaction that checkpoints its step.

**Incorrect (calling recv inside a step, sending without webhook idempotency, or looping single sends):**

```rust
use std::time::Duration;
use dbos::DBOS;

async fn bad_approval_step() -> dbos::Result<Option<String>> {
    // Fails with `Error::InsideStep`: `recv` consumes a message and MUST stand in the workflow body!
    dbos::step("wait_approval", || async {
        dbos::recv(Some("approvals"), Duration::from_secs(60)).await
    })
    .await
}

async fn retry_unsafe_webhook(
    dbos: &DBOS,
    workflow_id: &str,
    payload: String,
) -> dbos::Result<()> {
    // Duplicate webhook retries from Stripe/GitHub will insert duplicate messages with `idempotency_key: None`!
    dbos.send(workflow_id, &payload).await
}
```

**Correct (receiving by topic in workflow body, deduplicating webhooks, and batching with send_bulk):**

```rust
use std::time::Duration;
use dbos::{DBOS, Forks, SendMessage, SendOptions};

const APPROVAL_TOPIC: &str = "approval";

async fn expense_approval_workflow(report_id: String) -> dbos::Result<String> {
    dbos::step("notify_manager", || {
        let report_id = report_id.clone();
        async move {
            let _ = report_id;
            Ok(())
        }
    })
    .await?;

    // Parks durably up to 7 days; restarts resume with the remaining deadline!
    // Returns `Ok(None)` if the timeout elapses with no message on `Some(APPROVAL_TOPIC)`.
    let decision: Option<String> =
        dbos::recv(Some(APPROVAL_TOPIC), Duration::from_secs(7 * 24 * 3600)).await?;

    Ok(decision.unwrap_or_else(|| "expired".to_owned()))
}

// Deduplicate external HTTP/webhook deliveries with `SendMessage::idempotency_key`
async fn handle_approval_webhook(
    dbos: &DBOS,
    workflow_id: &str,
    webhook_delivery_id: &str,
    verdict: String,
) -> dbos::Result<()> {
    dbos.send_with(
        SendMessage {
            topic: Some(APPROVAL_TOPIC),
            idempotency_key: Some(webhook_delivery_id),
            ..SendMessage::new(workflow_id, &verdict)
        },
        SendOptions {
            send_to_forks: Forks::Skip,
            ..SendOptions::default()
        },
    )
    .await
}

// Deliver to multiple waiting workflows atomically (all-or-nothing) in one SQL insert
async fn approve_all_waiting(dbos: &DBOS, workflow_ids: &[String]) -> dbos::Result<()> {
    let verdict = "approved".to_owned();
    let batch: Vec<SendMessage<'_, String>> = workflow_ids
        .iter()
        .map(|id| SendMessage {
            topic: Some(APPROVAL_TOPIC),
            ..SendMessage::new(id, &verdict)
        })
        .collect();

    dbos.send_bulk(&batch).await
}
```

### Messaging Details and Edge Cases

- **Topic isolation (`Option<&str>`)**: `None` addresses the default topic, while `Some("approval")` addresses a named topic (`Some("")` is rejected with `Error::InvalidArgument`). A message sent on topic `"a"` is invisible to `recv(Some("b"), ..)` and `recv(None, ..)`.
- **Target workflow must already exist**: `dbos.notifications` enforces a foreign key against `workflow_status(workflow_uuid)`. Sending to a nonexistent `workflow_id` fails with `NonExistentWorkflow`, whereas sending before a workflow reaches `recv` (even while `ENQUEUED` or `DELAYED`) buffers the message safely until `recv` runs.
- **No concurrent `recv` on the same `(workflow_id, topic)`**: Racing two `recv` calls on the **same** topic inside `tokio::join!` or `select_step!` fails with `ConcurrentRecv`; use distinct topics when awaiting multiple message kinds concurrently.
- **Sending inside a step vs workflow body**: Inside a workflow body, `dbos::send` and `dbos::send_bulk` are checkpointed as steps (replays do not re-send). Inside a `dbos::step` body, `dbos::send_with` runs plainly, so pass `idempotency_key: Some(...)` if the step retries.
- **Fork fan-out**: Passing `SendOptions { send_to_forks: Forks::Include, .. }` delivers the message to `destination_id` and every workflow forked from it (transitively).

Reference: [dbos::recv](https://docs.rs/dbos/latest/dbos/fn.recv.html), [dbos::send_with](https://docs.rs/dbos/latest/dbos/fn.send_with.html), [dbos::send_bulk](https://docs.rs/dbos/latest/dbos/fn.send_bulk.html), [dbos::SendMessage](https://docs.rs/dbos/latest/dbos/struct.SendMessage.html)
