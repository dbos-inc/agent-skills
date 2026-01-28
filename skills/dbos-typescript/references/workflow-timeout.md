---
title: Set Workflow Timeouts
impact: CRITICAL
impactDescription: Prevents workflows from running indefinitely
tags: workflow, timeout, cancellation, duration
---

## Set Workflow Timeouts

Set a timeout for a workflow by passing `timeoutMS` to `DBOS.startWorkflow`. When the timeout expires, the workflow and all its children are cancelled.

**Incorrect (no timeout for potentially long workflow):**

```typescript
// No timeout - could run indefinitely
const handle = await DBOS.startWorkflow(processTask)("data");
```

**Correct (with timeout):**

```typescript
// Timeout after 5 minutes (in milliseconds)
const handle = await DBOS.startWorkflow(processTask, {
  timeoutMS: 5 * 60 * 1000,
})("data");
```

With decorators:

```typescript
class Tasks {
  @DBOS.workflow()
  static async processTask(data: string) {
    // ...
  }
}

const handle = await DBOS.startWorkflow(Tasks, {
  timeoutMS: 300000,
}).processTask("data");
```

Key timeout behaviors:
- Timeouts are **start-to-completion**: the timeout begins when the workflow starts execution, not when it's enqueued
- Timeouts are **durable**: they persist across restarts, so workflows can have very long timeouts (hours, days, weeks)
- Cancellation happens at the **beginning of the next step** - the current step completes first
- Cancelling a workflow also cancels all **child workflows**

Reference: [Workflow Timeouts](https://docs.dbos.dev/typescript/tutorials/workflow-tutorial#workflow-timeouts)
