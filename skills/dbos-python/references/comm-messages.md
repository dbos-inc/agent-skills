---
title: Use Messages for Workflow Notifications
impact: MEDIUM
impactDescription: Enables external signals to control workflow execution
tags: messages, send, recv, notifications, idempotency_key
---

## Use Messages for Workflow Notifications

Send messages to workflows to signal or notify them while running. Messages are persisted and queued per topic.

**Incorrect (polling external state):**

```python
@DBOS.workflow()
def payment_workflow():
    # Polling is inefficient and not durable
    while True:
        status = check_payment_status()
        if status == "paid":
            break
        time.sleep(1)
```

**Correct (using messages):**

```python
PAYMENT_STATUS = "payment_status"

@DBOS.workflow()
def payment_workflow():
    # Process order...
    DBOS.set_event("payment_id", payment_id)

    # Wait for payment notification (60 second timeout)
    payment_status = DBOS.recv(PAYMENT_STATUS, timeout_seconds=60)

    if payment_status == "paid":
        fulfill_order()
    else:
        cancel_order()

# Webhook endpoint to receive payment notification
@app.post("/payment_webhook/{workflow_id}/{status}")
def payment_webhook(workflow_id: str, status: str):
    DBOS.send(workflow_id, status, PAYMENT_STATUS)
    return {"ok": True}
```

Key points:
- `DBOS.recv()` can only be called from workflows
- Messages are queued per topic
- `recv()` returns `None` on timeout
- Messages are persisted

### Exactly-Once Sends with `idempotency_key`

Without an idempotency key, a `DBOS.send` called from outside a workflow (e.g. an HTTP handler, a retried webhook, a `DBOSClient`) may deliver the message more than once if the caller retries. Pass `idempotency_key` to deduplicate: any number of `send` calls with the same key will deliver the message exactly once.

**Incorrect (retried webhook delivers duplicate messages):**

```python
@app.post("/payment_webhook/{workflow_id}/{status}")
def payment_webhook(workflow_id: str, status: str):
    # If the webhook provider retries on a 5xx, the workflow receives
    # the same message multiple times.
    DBOS.send(workflow_id, status, PAYMENT_STATUS)
```

**Correct (exactly-once with idempotency_key):**

```python
@app.post("/payment_webhook/{workflow_id}/{status}")
def payment_webhook(workflow_id: str, status: str, request: Request):
    # Use a stable key from the webhook payload (e.g. event ID) so retries
    # of the same logical event are deduplicated.
    event_id = request.headers.get("X-Event-Id")
    DBOS.send(
        workflow_id,
        status,
        PAYMENT_STATUS,
        idempotency_key=event_id,
    )
```

The same parameter is available on `DBOS.send_async` and `client.send` / `client.send_async`. Strongly recommended whenever calling `send` from outside a workflow. The idempotency key is scoped per destination workflow, so the same key sent to different destinations delivers to each.

### Sending to Forks with `send_to_forks`

`DBOS.send` / `DBOS.send_async` (and `client.send` / `client.send_async`) accept `send_to_forks: bool = False`. If `True`, also deliver the message to every workflow recursively forked from `destination_id`. Defaults to `False`.

```python
DBOS.send(workflow_id, status, PAYMENT_STATUS, send_to_forks=True)
```

### Atomic Multi-Send with `send_bulk`

`DBOS.send_bulk(messages: List[SendMessage], *, send_to_forks: bool = False) -> None` sends many messages in a single atomic transaction: if any message can't be delivered, the entire batch rolls back and nothing is sent. Also available as `DBOS.send_bulk_async`, `client.send_bulk`, and `client.send_bulk_async`.

Each message is a `SendMessage` dataclass:

```python
@dataclass
class SendMessage:
    destination_id: str            # workflow to send to
    message: Any                   # must be serializable
    topic: Optional[str] = None
    idempotency_key: Optional[str] = None
```

```python
from dbos import SendMessage

DBOS.send_bulk([
    SendMessage(workflow_a, "paid", PAYMENT_STATUS),
    SendMessage(workflow_b, "paid", PAYMENT_STATUS),
])
```

- Two messages in the same call may not share an idempotency key.
- Use `send_bulk_async` (not `send_bulk`) inside coroutine workflows.

Reference: [Workflow Messaging](https://docs.dbos.dev/python/tutorials/workflow-communication#workflow-messaging-and-notifications)
