---
title: Start Workflows in Background
impact: CRITICAL
impactDescription: Background workflows enable reliable async processing
tags: workflow, background, handle, async
---

## Start Workflows in Background

Use `DBOS.startWorkflow` to start a workflow in the background and get a handle to track it. The workflow is guaranteed to run to completion even if the app is interrupted.

**Incorrect (no way to track background work):**

```typescript
class Example {
  @DBOS.workflow()
  static async processData(data: string) {
    // ...
  }
}

// Fire and forget - no way to track or get result
Example.processData(data);
```

**Correct (using startWorkflow):**

```typescript
class Example {
  @DBOS.workflow()
  static async processData(data: string) {
    return "processed: " + data;
  }
}

async function main() {
  // Start workflow in background, get handle
  const handle = await DBOS.startWorkflow(Example).processData("input");

  // Get the workflow ID
  console.log(handle.workflowID);

  // Wait for result
  const result = await handle.getResult();

  // Check status
  const status = await handle.getStatus();
}
```

Or with `registerWorkflow`:

```typescript
async function processDataFn(data: string) {
  return "processed: " + data;
}
const processData = DBOS.registerWorkflow(processDataFn);

const handle = await DBOS.startWorkflow(processData)("input");
const result = await handle.getResult();
```

Retrieve a handle later by workflow ID:

```typescript
const handle = DBOS.retrieveWorkflow<string>(workflowID);
const result = await handle.getResult();
```

Reference: [Starting Workflows in Background](https://docs.dbos.dev/typescript/tutorials/workflow-tutorial#starting-workflows-in-the-background)
