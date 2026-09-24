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
const workflowID = `payment-${orderId}`;
await DBOS.startWorkflow(processPayment, { workflowID })("order-123", 50);
await DBOS.startWorkflow(processPayment, { workflowID })("order-123", 50);
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

By default, starting a workflow with an ID already in use (whatever its status) returns a handle to the existing workflow. To throw `DBOSWorkflowIDInUseError` instead, pass `workflowIDReusePolicy: 'reject'` to `DBOS.startWorkflow` (DBOS 5.1+); match the error with `isWorkflowIDInUseError` from the SDK's `Error` namespace rather than `instanceof`.

Reference: [Workflow IDs and Idempotency](https://docs.dbos.dev/typescript/tutorials/workflow-tutorial#workflow-ids-and-idempotency)
