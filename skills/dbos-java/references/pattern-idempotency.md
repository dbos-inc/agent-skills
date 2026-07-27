---
title: Use Workflow IDs as Idempotency Keys
impact: MEDIUM
impactDescription: Prevents duplicate execution of operations with side effects
tags: pattern, idempotency, workflow-id, deduplication, retry
---

## Use Workflow IDs as Idempotency Keys

Every workflow execution has a globally unique ID, by default a UUID. Setting the ID explicitly makes the workflow
idempotent: repeated calls with the same ID resolve to the single original execution rather than starting a new one.
This is the standard way to make retried HTTP requests safe.

**Incorrect (guarding with an application-level check):**

```java
// A retry that arrives before the first request commits still double-charges
if (!orderRepo.exists(requestId)) {
  dbos.startWorkflow(() -> proxy.checkout(requestId));
}
```

**Correct (explicit workflow ID as the idempotency key):**

```java
app.post("/checkout/{idempotencyKey}", ctx -> {
  String key = ctx.pathParam("idempotencyKey");

  // Calling this repeatedly with the same key executes the workflow exactly once
  WorkflowHandle<String, Exception> handle = dbos.startWorkflow(
      () -> checkoutProxy.checkout(key),
      new StartWorkflowOptions().withWorkflowId(key));

  ctx.result(handle.getResult());
});
```

Details:

- Workflow IDs are globally unique per application; reusing an ID returns a handle to the existing execution
  instead of starting new work
- Read the current ID inside a workflow or step with `DBOS.workflowId()`
- Use a natural key from the caller (request ID, order ID, `"sync-" + userId + "-" + date`) so retries produce the
  same ID
- For workflows invoked directly rather than through `startWorkflow`, set the ID on the calling context:

```java
try (var opts = new WorkflowOptions("order-123").setContext()) {
  proxy.checkout("order-123");
}
```

- Child workflows started inside a workflow are already exactly-once; explicit IDs are for entry points
- To reject rather than coalesce concurrent duplicates on a queue, use a deduplication ID
  ([queue-deduplication.md](queue-deduplication.md))

Reference: [Workflow IDs and Idempotency](https://docs.dbos.dev/java/tutorials/workflow-tutorial#workflow-ids-and-idempotency)
