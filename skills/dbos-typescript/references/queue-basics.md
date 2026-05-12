---
title: Use Queues for Concurrent Workflows
impact: HIGH
impactDescription: Queues provide managed concurrency and flow control
tags: queue, concurrency, enqueue, workflow, registerQueue
---

## Use Queues for Concurrent Workflows

Queues run many workflows concurrently with managed flow control. Use them when you need to control how many workflows run at once.

Register queues with `DBOS.registerQueue` **after** `DBOS.launch()`. Queue configuration is persisted to the system database, so all DBOS processes and clients connected to the same system database see it.

**Incorrect (uncontrolled concurrency):**

```typescript
async function processTaskFn(task: string) {
  // ...
}
const processTask = DBOS.registerWorkflow(processTaskFn);

// Starting many workflows without control - could overwhelm resources
for (const task of tasks) {
  await DBOS.startWorkflow(processTask)(task);
}
```

**Incorrect (deprecated in-memory `WorkflowQueue` constructor):**

```typescript
// Deprecated: in-memory only, not visible to other processes or clients
const queue = new WorkflowQueue("task_queue");
```

**Correct (database-backed queue):**

```typescript
import { DBOS } from "@dbos-inc/dbos-sdk";

async function processTaskFn(task: string) {
  // ...
}
const processTask = DBOS.registerWorkflow(processTaskFn);

async function processAllTasksFn(tasks: string[]) {
  const handles = [];
  for (const task of tasks) {
    // Enqueue by passing queueName to startWorkflow
    const handle = await DBOS.startWorkflow(processTask, {
      queueName: "task_queue",
    })(task);
    handles.push(handle);
  }
  const results = [];
  for (const h of handles) {
    results.push(await h.getResult());
  }
  return results;
}
const processAllTasks = DBOS.registerWorkflow(processAllTasksFn);

async function main() {
  await DBOS.launch();
  // Register queues AFTER launch
  await DBOS.registerQueue("task_queue");
}
```

`DBOS.registerQueue` returns a `WorkflowQueue` you can also use directly:

```typescript
const queue = await DBOS.registerQueue("task_queue");
const handle = await DBOS.startWorkflow(processTask, { queueName: queue.name })(task);
```

Queues process workflows in FIFO order.

`onConflict` controls how `registerQueue` handles an existing queue in the system database:
- `'update_if_latest_version'` (default): overwrite only if this app is the latest registered application version
- `'always_update'`: always overwrite
- `'never_update'`: leave existing configuration unchanged (use this if you reconfigured the queue at runtime via `set` methods)

Reference: [DBOS Queues](https://docs.dbos.dev/typescript/tutorials/queue-tutorial)
