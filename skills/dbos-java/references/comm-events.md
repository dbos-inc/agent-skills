---
title: Publish Workflow Progress with Events
impact: MEDIUM
impactDescription: Lets callers read workflow state before the workflow completes
tags: communication, events, setEvent, getEvent, interactive
---

## Publish Workflow Progress with Events

Events are key-value pairs a workflow publishes about itself. Callers read them with `getEvent`, which waits until
the key is published or the timeout expires. Use events to return intermediate results — a payment URL, a job ID —
without waiting for the whole workflow.

**Incorrect (polling a side table for progress):**

```java
// Extra table, extra writes, and no built-in waiting
dbos.runStep(() -> statusRepo.save(workflowId, "payment-id", paymentId), "saveStatus");
```

**Correct (setEvent in the workflow, getEvent in the caller):**

```java
class CheckoutImpl implements Checkout {
  private static final String PAYMENT_ID = "payment_id";
  private final DBOS dbos;

  CheckoutImpl(DBOS dbos) { this.dbos = dbos; }

  @Override
  @Workflow
  public void checkoutWorkflow() {
    String paymentId = dbos.runStep(this::createPayment, "createPayment");
    dbos.setEvent(PAYMENT_ID, paymentId); // publish, or update if already published
    // ... continue processing
  }
}

// Caller: start the workflow, then wait for the event
var handle = dbos.startWorkflow(() -> checkoutProxy.checkoutWorkflow(),
    new StartWorkflowOptions().withWorkflowId(idempotencyKey));

Optional<String> paymentId =
    dbos.<String>getEvent(handle.workflowId(), PAYMENT_ID, Duration.ofSeconds(60));
```

API and behavior:

- `setEvent(String key, Object value)` may only be called from inside a workflow; calling it again for the same key
  overwrites the value
- `getEvent(String workflowId, String key, Duration timeout)` returns `Optional.empty()` on timeout, and may be
  called from a workflow, from ordinary code, or from `DBOSClient`
- Events are persisted, so the latest value is always retrievable after the workflow finishes
- When `getEvent` is called inside a workflow, the retrieved value is checkpointed so replay sees the same value
  even if the event later changes
- `dbos.getAllEvents(workflowId)` returns every published key as a `Map<String, Object>`
- Values must be JSON-serializable; pass `SerializationStrategy.PORTABLE` to `setEvent` when a Python or TypeScript
  application will read the event ([advanced-interops.md](advanced-interops.md))
- If the workflow is cancelled while waiting in `getEvent`, `DBOSWorkflowCancelledException` is thrown

Reference: [Workflow Events](https://docs.dbos.dev/java/tutorials/workflow-communication#workflow-events)
