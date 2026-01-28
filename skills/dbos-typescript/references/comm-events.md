---
title: Use Events for Workflow Status Publishing
impact: MEDIUM
impactDescription: Enables real-time progress monitoring and interactive workflows
tags: communication, events, status, key-value
---

## Use Events for Workflow Status Publishing

Workflows can publish events (key-value pairs) with `DBOS.setEvent`. Other code can read events with `DBOS.getEvent`. Events are persisted and useful for real-time progress monitoring.

**Incorrect (using external state for progress):**

```typescript
let progress = 0; // Global variable - not durable!

class Example {
  @DBOS.workflow()
  static async processData() {
    progress = 50; // Not persisted, lost on restart
  }
}
```

**Correct (using events):**

```typescript
class Example {
  @DBOS.workflow()
  static async processData() {
    await DBOS.setEvent("status", "processing");
    await Example.stepOne();
    await DBOS.setEvent("progress", 50);
    await Example.stepTwo();
    await DBOS.setEvent("progress", 100);
    await DBOS.setEvent("status", "complete");
  }
}

// Read events from outside the workflow
const status = await DBOS.getEvent<string>(workflowID, "status");
const progress = await DBOS.getEvent<number>(workflowID, "progress", 30);
// Returns null if the event doesn't exist within the timeout (default 60s)
```

Events are useful for interactive workflows. For example, a checkout workflow can publish a payment URL for the caller to redirect to:

```typescript
class Shop {
  @DBOS.workflow()
  static async checkoutWorkflow() {
    const paymentURL = await Shop.createPayment();
    await DBOS.setEvent("paymentURL", paymentURL);
    // Continue processing...
  }
}

// HTTP handler starts workflow and reads the payment URL
const handle = await DBOS.startWorkflow(Shop).checkoutWorkflow();
const url = await DBOS.getEvent<string>(handle.workflowID, "paymentURL", 300);
```

Reference: [Workflow Events](https://docs.dbos.dev/typescript/tutorials/workflow-communication#workflow-events)
