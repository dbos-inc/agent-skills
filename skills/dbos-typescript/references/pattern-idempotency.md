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
class Payments {
  @DBOS.workflow()
  static async processPayment(orderId: string, amount: number) {
    await Payments.chargeCard(amount);
    await Payments.updateOrder(orderId);
  }
}

// Multiple calls could charge the card multiple times!
await Payments.processPayment("order-123", 50);
await Payments.processPayment("order-123", 50); // Double charge!
```

**Correct (with workflow ID):**

```typescript
class Payments {
  @DBOS.workflow()
  static async processPayment(orderId: string, amount: number) {
    await Payments.chargeCard(amount);
    await Payments.updateOrder(orderId);
  }
}

// Same workflow ID = only one execution
const workflowID = `payment-${orderId}`;
await DBOS.startWorkflow(Payments, { workflowID }).processPayment("order-123", 50);
await DBOS.startWorkflow(Payments, { workflowID }).processPayment("order-123", 50);
// Second call returns the result of the first execution
```

Or with `registerWorkflow`:

```typescript
async function processPaymentFn(orderId: string, amount: number) {
  // ...
}
const processPayment = DBOS.registerWorkflow(processPaymentFn);

const handle = await DBOS.startWorkflow(processPayment, {
  workflowID: `payment-${orderId}`,
})(orderId, amount);
```

Access the current workflow ID inside a workflow:

```typescript
async function myWorkflowFn() {
  const currentID = DBOS.workflowID;
  console.log(`Running workflow: ${currentID}`);
}
```

Workflow IDs must be **globally unique** for your application. If not set, a random UUID is generated.

Reference: [Workflow IDs and Idempotency](https://docs.dbos.dev/typescript/tutorials/workflow-tutorial#workflow-ids-and-idempotency)
