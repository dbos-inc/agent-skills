---
title: Use Messages for Workflow Notifications
impact: MEDIUM
impactDescription: Enables reliable inter-workflow and external-to-workflow communication
tags: communication, messages, send, recv, notification, RecvOptions
---

## Use Messages for Workflow Notifications

Use `DBOS.send` to send messages to a workflow and `DBOS.recv` to receive them. Messages are queued per topic and persisted for reliable delivery.

**Incorrect (using external messaging for workflow communication):**

```typescript
// External message queue is not integrated with workflow recovery
import { Queue } from "some-external-queue";
```

**Correct (using DBOS messages):**

```typescript
async function checkoutWorkflowFn() {
  // Wait for payment notification (default 60s timeout).
  // Pass options as an object (the deprecated positional timeoutSeconds form
  // still works but is no longer the recommended call shape).
  const notification = await DBOS.recv<string>("payment_status", {
    timeoutSeconds: 120,
  });

  if (notification && notification === "paid") {
    await DBOS.runStep(fulfillOrder, { name: "fulfillOrder" });
  } else {
    await DBOS.runStep(cancelOrder, { name: "cancelOrder" });
  }
}
const checkoutWorkflow = DBOS.registerWorkflow(checkoutWorkflowFn);

// Send a message from a webhook handler
async function paymentWebhook(workflowID: string, status: string) {
  await DBOS.send(workflowID, status, "payment_status");
}
```

### `recv` Options

```typescript
interface RecvOptions {
  timeoutSeconds?: number;     // Relative timeout from now (default 60)
  deadlineEpochMS?: number;    // Absolute deadline as Unix epoch ms
  pollingIntervalMs?: number;  // Interval between system-database polls while waiting
}
```

Use `timeoutSeconds` for a relative wait, or `deadlineEpochMS` to pin to an absolute clock time (useful when the workflow has its own deadline):

```typescript
// Wait up to 30 seconds
const msg = await DBOS.recv<string>("topic", { timeoutSeconds: 30 });

// Wait until a specific wall-clock time
const deadline = Date.now() + 10_000;
const msg2 = await DBOS.recv<string>("topic", { deadlineEpochMS: deadline });

// Poll once without waiting
const snapshot = await DBOS.recv<string>("topic", { timeoutSeconds: 0 });
```

Key behaviors:
- `recv` waits for and consumes the next message for the specified topic
- Returns `null` if the wait times out
- Messages without a topic can only be received by `recv` without a topic
- Messages are queued per-topic (FIFO)

### Reliability Guarantees

- All messages are persisted to the database
- Messages sent from workflows are delivered exactly-once
- Messages sent from non-workflow code (HTTP handlers, retried webhooks, `DBOSClient`) can use an idempotency key for exactly-once delivery:

```typescript
// Use a stable key from the event source so webhook retries dedupe
await DBOS.send(workflowID, message, "topic", eventId);
```

Reference: [Workflow Messaging](https://docs.dbos.dev/typescript/tutorials/workflow-communication#workflow-messaging-and-notifications)
