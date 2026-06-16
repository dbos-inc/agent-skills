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

async function processDataFn() {
  progress = 50; // Not persisted, lost on restart
}
const processData = DBOS.registerWorkflow(processDataFn);
```

**Correct (using events):**

```typescript
async function processDataFn() {
  await DBOS.setEvent("status", "processing");
  await DBOS.runStep(stepOne, { name: "stepOne" });
  await DBOS.setEvent("progress", 50);
  await DBOS.runStep(stepTwo, { name: "stepTwo" });
  await DBOS.setEvent("progress", 100);
  await DBOS.setEvent("status", "complete");
}
const processData = DBOS.registerWorkflow(processDataFn);

// Read events from outside the workflow.
// Pass options as an object (the deprecated positional timeoutSeconds form
// still works but is no longer the recommended call shape).
const status = await DBOS.getEvent<string>(workflowID, "status", {
  timeoutSeconds: 5,
});
const progress = await DBOS.getEvent<number>(workflowID, "progress");
// Returns null if the event doesn't exist within the timeout (default 60s)
```

### `getEvent` Options

```typescript
interface GetEventOptions {
  timeoutSeconds?: number;     // Relative timeout from now (default 60)
  deadlineEpochMS?: number;    // Absolute deadline as Unix epoch ms
  pollingIntervalMs?: number;  // Interval between system-database polls while waiting
}
```

Use `timeoutSeconds` for a relative wait, or `deadlineEpochMS` to pin to an absolute clock time (useful when the caller has its own request deadline):

```typescript
// Wait up to 30 seconds
const status = await DBOS.getEvent<string>(workflowID, "status", {
  timeoutSeconds: 30,
});

// Wait until a specific wall-clock time (e.g., the HTTP request deadline)
const requestDeadline = Date.now() + 10_000;
const status2 = await DBOS.getEvent<string>(workflowID, "status", {
  deadlineEpochMS: requestDeadline,
});

// Poll once without waiting (timeout of 0)
const snapshot = await DBOS.getEvent<string>(workflowID, "status", {
  timeoutSeconds: 0,
});
```

### Interactive Workflows

Events are useful for interactive workflows. For example, a checkout workflow can publish a payment URL for the caller to redirect to:

```typescript
async function checkoutWorkflowFn() {
  const paymentURL = await DBOS.runStep(createPayment, { name: "createPayment" });
  await DBOS.setEvent("paymentURL", paymentURL);
  // Continue processing...
}
const checkoutWorkflow = DBOS.registerWorkflow(checkoutWorkflowFn);

// HTTP handler starts workflow and waits up to 5 minutes for the payment URL
const handle = await DBOS.startWorkflow(checkoutWorkflow)();
const url = await DBOS.getEvent<string>(handle.workflowID, "paymentURL", {
  timeoutSeconds: 300,
});
```

Reference: [Workflow Events](https://docs.dbos.dev/typescript/tutorials/workflow-communication#workflow-events)
