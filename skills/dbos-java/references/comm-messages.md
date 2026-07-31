---
title: Signal Running Workflows with Messages
impact: MEDIUM
impactDescription: Exactly-once notifications let workflows wait durably for external events
tags: communication, messages, send, recv, notifications, webhook
---

## Signal Running Workflows with Messages

`send` delivers a message to a specific workflow; `recv` consumes the next message for a topic, waiting durably up
to a timeout. This is how a workflow waits for a webhook, an approval, or a child's completion signal without
polling.

**Incorrect (polling a database table for the signal):**

```java
@Workflow
public void checkoutWorkflow() throws Exception {
  while (true) {
    String status = dbos.runStep(() -> paymentRepo.status(orderId), "checkStatus");
    if (status != null) break;
    dbos.sleep(Duration.ofSeconds(5)); // one checkpoint per poll, forever
  }
}
```

**Correct (recv in the workflow, send from the handler):**

```java
class CheckoutImpl implements Checkout {
  private static final String PAYMENT_STATUS = "payment_status";
  private final DBOS dbos;

  CheckoutImpl(DBOS dbos) { this.dbos = dbos; }

  @Override
  @Workflow
  public void checkoutWorkflow() {
    // Redirect the customer to a payment page, then wait durably for the result
    Optional<String> status = dbos.<String>recv(PAYMENT_STATUS, Duration.ofMinutes(5));
    if (status.filter("paid"::equals).isPresent()) {
      dbos.runStep(this::fulfillOrder, "fulfillOrder");
    } else {
      dbos.runStep(this::cancelOrder, "cancelOrder");
    }
  }
}

// Webhook handler — outside the workflow
dbos.send(workflowId, paymentStatus, PAYMENT_STATUS, idempotencyKey);
```

API and behavior:

- `send(String destinationId, Object message, String topic)` and an overload taking an `idempotencyKey`; when
  sending from ordinary (non-workflow) code, supply the key to get exactly-once delivery
- Sends made from inside a workflow are exactly-once automatically
- `recv(String topic, Duration timeout)` may only be called from a workflow; it consumes messages FIFO per topic and
  returns `Optional.empty()` on timeout. Passing `null` as the topic receives only messages sent without a topic.
- Messages are persisted before `send` returns, so a message sent to a workflow that is not yet at its `recv` is
  still delivered
- `sendBulk(List<SendMessage>)` sends a batch in one call; destinations may differ:

```java
dbos.sendBulk(List.of(
    new SendMessage(orderWorkflowId, "confirmed", ORDER_STATUS),
    new SendMessage(inventoryWorkflowId, order, RESERVE_TOPIC)));
```

- `DBOSClient.send` and `DBOSClient.sendBulk` do the same from outside the application
- If the workflow is cancelled while waiting in `recv`, `DBOSWorkflowCancelledException` is thrown

Reference: [Workflow Messaging](https://docs.dbos.dev/java/tutorials/workflow-communication#workflow-messaging-and-notifications)
