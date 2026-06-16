---
title: Start Workflows in Background
impact: CRITICAL
impactDescription: Background workflows enable reliable async processing
tags: workflow, background, handle, async, waitFirst
---

## Start Workflows in Background

Use `DBOS.startWorkflow` to start a workflow in the background and get a handle to track it. The workflow is guaranteed to run to completion even if the app is interrupted.

**Incorrect (no way to track background work):**

```typescript
async function processDataFn(data: string) {
  // ...
}
const processData = DBOS.registerWorkflow(processDataFn);

// Fire and forget - no way to track or get result
processData(data);
```

**Correct (using startWorkflow):**

```typescript
async function processDataFn(data: string) {
  return "processed: " + data;
}
const processData = DBOS.registerWorkflow(processDataFn);

async function main() {
  // Start workflow in background, get handle
  const handle = await DBOS.startWorkflow(processData)("input");

  // Get the workflow ID
  console.log(handle.workflowID);

  // Wait for result
  const result = await handle.getResult();

  // Check status
  const status = await handle.getStatus();
}
```

Retrieve a handle later by workflow ID:

```typescript
const handle = DBOS.retrieveWorkflow<string>(workflowID);
const result = await handle.getResult();

// getResult accepts options?: { pollingIntervalMs?: number } to set the
// interval between system-database polls. This only applies to handles that
// wait by polling the DB (e.g. from DBOS.retrieveWorkflow or the DBOS Client),
// not to a handle from DBOS.startWorkflow in the same process.
const slowPoll = await handle.getResult({ pollingIntervalMs: 5000 });
```

### Waiting for the First of Multiple Workflows

Use `DBOS.waitFirst` to race multiple concurrent workflows and process results as they complete:

```typescript
const handles = await Promise.all(
  items.map((item) => DBOS.startWorkflow(processItem)(item))
);

// Wait for whichever finishes first
const firstDone = await DBOS.waitFirst(handles);
const result = await firstDone.getResult();
```

`waitFirst` takes a non-empty array of `WorkflowHandle` and throws if the array is empty. It accepts `DBOS.waitFirst(handles, options?: { pollingIntervalMs?: number })` to set the interval between system-database polls while waiting.

Reference: [Starting Workflows in Background](https://docs.dbos.dev/typescript/tutorials/workflow-tutorial#starting-workflows-in-the-background)
