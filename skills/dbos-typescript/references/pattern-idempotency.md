---
title: Use Workflow IDs for Idempotency
impact: MEDIUM
impactDescription: Prevents duplicate side effects like double payments
tags: pattern, idempotency, workflow-id, deduplication
---

## Use Workflow IDs for Idempotency

Assign a workflow ID to ensure a workflow executes only once, even if called multiple times. This prevents duplicate side effects like double payments.

**Incorrect (no idempotency):**

```typescript
async function processPaymentFn(orderId: string, amount: number) {
  await DBOS.runStep(() => chargeCard(amount), { name: "chargeCard" });
  await DBOS.runStep(() => updateOrder(orderId), { name: "updateOrder" });
}
const processPayment = DBOS.registerWorkflow(processPaymentFn);

// Multiple calls could charge the card multiple times!
await processPayment("order-123", 50);
await processPayment("order-123", 50); // Double charge!
```

**Correct (with workflow ID):**

```typescript
async function processPaymentFn(orderId: string, amount: number) {
  await DBOS.runStep(() => chargeCard(amount), { name: "chargeCard" });
  await DBOS.runStep(() => updateOrder(orderId), { name: "updateOrder" });
}
const processPayment = DBOS.registerWorkflow(processPaymentFn);

// Same workflow ID = only one execution
const orderId = "order-123";
const workflowID = `payment-${orderId}`;
await DBOS.startWorkflow(processPayment, { workflowID })(orderId, 50);
await DBOS.startWorkflow(processPayment, { workflowID })(orderId, 50);
// Second call does not start a new workflow; it returns a handle to the existing one
```

Access the current workflow ID inside a workflow:

```typescript
async function myWorkflowFn() {
  const currentID = DBOS.workflowID;
  console.log(`Running workflow: ${currentID}`);
}
```

Workflow IDs must be **globally unique** for your application (and across all applications sharing the system database). If not set, a random UUID is generated (child workflows started from a workflow get a deterministic ID derived from the parent's ID).

### Rejecting Reused Workflow IDs

`workflowIDReusePolicy` controls what happens when a workflow with the given ID already exists, whatever its status:
- `'return-existing'` (default): return a handle to the existing workflow without starting a new one
- `'reject'`: throw `DBOSWorkflowIDInUseError` without starting a new workflow or modifying the existing one. The error has `workflowID`, `status`, and `workflowName` properties

Match the error with `isWorkflowIDInUseError` from the SDK's `Error` namespace, not `instanceof` (an error replayed from a workflow's recorded history may not be an instance of the class):

```typescript
import { DBOS, Error as DBOSErrors } from "@dbos-inc/dbos-sdk";

// processOrder is a registered workflow

async function submitOrder(orderID: string, order: Order) {
  try {
    const handle = await DBOS.startWorkflow(processOrder, {
      workflowID: `order-${orderID}`,
      workflowIDReusePolicy: "reject",
    })(order);
    return await handle.getResult();
  } catch (e) {
    if (DBOSErrors.isWorkflowIDInUseError(e)) {
      return "already started";
    }
    throw e;
  }
}
```

The same option is available on `client.enqueue` (and `DBOS.enqueueWorkflowWithOptions`):

```typescript
// client is a DBOSClient; orderID and order as in submitOrder above
await client.enqueue(
  { workflowName: "processOrder", queueName: "orders", workflowID: `order-${orderID}`, workflowIDReusePolicy: "reject" },
  order,
);
```

`workflowIDReusePolicy` is about the workflow ID; `duplicationPolicy` is about queue deduplication IDs (see `queue-deduplication.md`).

Reference: [Workflow IDs and Idempotency](https://docs.dbos.dev/typescript/tutorials/workflow-tutorial#workflow-ids-and-idempotency)
