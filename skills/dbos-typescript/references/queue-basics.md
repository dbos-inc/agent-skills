---
title: Use Queues for Concurrent Workflows
impact: HIGH
impactDescription: Queues provide managed concurrency and flow control
tags: queue, concurrency, enqueue, workflow, registerQueue, priority
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

**Incorrect (in-memory `WorkflowQueue` constructor, removed in 5.0):**

```typescript
// Removed in DBOS 5.0 - use DBOS.registerQueue after launch instead
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
  DBOS.setConfig({ name: "my-app", applicationVersion: "0.1.0", systemDatabaseUrl: process.env.DBOS_SYSTEM_DATABASE_URL });
  await DBOS.launch();
  // Register queues AFTER launch
  await DBOS.registerQueue("task_queue");
}
```

Queues process workflows in FIFO order (within a priority level, see below).

Key behaviors:
- Register every queue you enqueue on. A workflow enqueued on a queue that isn't registered stays `ENQUEUED` until a queue with that name is registered.
- Queue names must be unique within the system database (including queues of other applications sharing it). Names starting with `_dbos_` are reserved.
- Queues are owned by the application (its configured `name`) that registers them. A process dequeues from all queues owned by its application (see `listenQueues` to restrict this).

`onConflict` controls how `registerQueue` handles an existing queue in the system database:
- `'update_if_latest_version'` (default): overwrite only if this app is the latest registered application version
- `'always_update'`: always overwrite
- `'never_update'`: leave existing configuration unchanged (use this if you reconfigured the queue at runtime via `set` methods)

### Priority

Priority is enabled on every queue; no configuration is needed. Set `priority` in `enqueueOptions`:

```typescript
// High priority task (lower number = higher priority)
await DBOS.startWorkflow(processTask, {
  queueName: "task_queue",
  enqueueOptions: { priority: 1 },
})("urgent-task");

// Low priority task
const handle = await DBOS.startWorkflow(processTask, {
  queueName: "task_queue",
  enqueueOptions: { priority: 100 },
})("background-task");
```

- Range: `0` to `2,147,483,647`; lower number = higher priority
- Workflows without an assigned priority have priority `0`, the highest
- Workflows with the same priority are dequeued in FIFO order
- The `priorityEnabled` queue option was removed in 5.0

Change the priority of a workflow that is still `ENQUEUED` or `DELAYED`:

```typescript
await DBOS.setWorkflowPriority(handle.workflowID, 0); // Promote to highest priority
```

Throws `DBOSInvalidQueuePriorityError` if the priority is out of range.

Reference: [DBOS Queues](https://docs.dbos.dev/typescript/tutorials/queue-tutorial)
