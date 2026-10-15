---
title: Use Messages for Workflow Notifications
impact: MEDIUM
impactDescription: Enables reliable inter-workflow and external-to-workflow communication
tags: communication, messages, send, recv, notification, bulk, forks
---

## Use Messages for Workflow Notifications

Use `dbos.Send` to send messages to a workflow and `dbos.Recv` to receive them. Messages are queued per topic and persisted for reliable delivery.

**Incorrect (using external messaging for workflow communication):**

```go
// External message queue is not integrated with workflow recovery
ch := make(chan string) // Not durable!
```

**Correct (using DBOS messages):**

```go
func checkoutWorkflow(ctx dbos.Context, orderID string) (string, error) {
	// Wait for payment notification (timeout 120 seconds)
	notification, err := dbos.Recv[string](ctx, "payment_status", 120*time.Second)
	if err != nil {
		return "", err
	}

	if notification == "paid" {
		_, err = dbos.RunAsStep(ctx, func(ctx context.Context) (string, error) {
			return fulfillOrder(orderID)
		}, dbos.WithStepName("fulfillOrder"))
		return "fulfilled", err
	}
	_, err = dbos.RunAsStep(ctx, func(ctx context.Context) (string, error) {
		return cancelOrder(orderID)
	}, dbos.WithStepName("cancelOrder"))
	return "cancelled", err
}

// Send a message from a webhook handler
func paymentWebhook(ctx dbos.Context, workflowID, status string) error {
	return dbos.Send(ctx, workflowID, status, "payment_status")
}
```

Key behaviors:
- `Recv` waits for and consumes the next message for the specified topic; it can only be called inside a workflow
- Returns the zero value if the wait times out, with a `dbos.Error` with code `ErrorCodeTimeout` (matches `dbos.ErrTimeout`)
- `Send` takes a `dbos.Client`, so it also works from external applications using the DBOS Client (a `dbos.Context` satisfies `Client`)
- Messages without a topic can only be received by `Recv` without a topic
- Messages are queued per-topic (FIFO)

**Reliability guarantees:**
- All messages are persisted to the database
- Messages sent from workflows are delivered exactly-once
- Outside a workflow, pass `WithIdempotencyKey(key)` so a retried `Send` delivers at most once: the key is combined with the destination workflow ID, so retrying with the same key inserts the message only once

### Sending to forks

Pass `WithSendToForks()` to also deliver the message to every workflow recursively forked from the destination (forks, forks of forks, and so on) that exists at send time (v1.6.0+):

```go
err := dbos.Send(ctx, workflowID, status, "payment_status", dbos.WithSendToForks())
```

### Atomic multi-send with `SendBulk`

`dbos.SendBulk` sends many messages, possibly to different workflows, in a single transaction: either every message is delivered or none is (v1.3.0+). Inside a workflow the whole batch is checkpointed as one step, so it is delivered exactly once. A batch may carry at most `dbos.MaxSendBulkMessages` (10,000) messages:

```go
err := dbos.SendBulk(ctx, []dbos.SendMessage{
	{DestinationID: orderWorkflowID, Message: "confirmed", Topic: "orders"},
	{DestinationID: inventoryWorkflowID, Message: order, Topic: "reserve", IdempotencyKey: "reserve-42"},
})
```

`SendMessage` fields: `DestinationID`, `Message` (must be serializable), `Topic` (empty sends without a topic), `IdempotencyKey`. Two messages in the same call may not share an idempotency key, and `WithIdempotencyKey` is rejected on `SendBulk`: set it per message instead. `WithSendToForks()` and `WithPortableSend()` apply to the whole batch.

### Sending in your own transaction

From a standalone client or a context outside a workflow, pass an open transaction on the DBOS system database (a `pgx.Tx`, a `*sql.Tx`, or a `dbos.Tx`) with `WithSendTransaction(tx)` so the message commits atomically with your own writes. You own the transaction: the message is invisible until you commit, and a failed send leaves the transaction aborted, so roll it back rather than retrying on it. Not available inside a workflow, where a send is checkpointed as a step:

```go
tx, _ := pool.Begin(ctx)
defer tx.Rollback(ctx)
tx.Exec(ctx, "INSERT INTO orders (id) VALUES ($1)", orderID)
err := dbos.Send(client, workflowID, orderID, "orders", dbos.WithSendTransaction(tx))
tx.Commit(ctx)
```

Reference: [Workflow Messaging and Notifications](https://docs.dbos.dev/golang/tutorials/workflow-communication#workflow-messaging-and-notifications)
