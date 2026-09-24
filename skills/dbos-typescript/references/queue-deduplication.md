---
title: Deduplicate Queued Workflows
impact: HIGH
impactDescription: Prevents duplicate workflow executions
tags: queue, deduplication, idempotent, duplicate, singleton
---

## Deduplicate Queued Workflows

Set a deduplication ID when enqueuing to prevent duplicate workflow executions. If a workflow with the same deduplication ID is already delayed, enqueued, or executing on the queue, a `DBOSQueueDuplicatedError` is thrown.

**Incorrect (no deduplication):**

```typescript
// Multiple clicks could enqueue duplicates
async function handleClick(userId: string) {
  await DBOS.startWorkflow(processTask, { queueName: "task_queue" })("task");
}
```

**Correct (with deduplication):**

```typescript
async function processTaskFn(task: string) {
  // ...
}
const processTask = DBOS.registerWorkflow(processTaskFn);

// After DBOS.launch()
await DBOS.registerQueue("task_queue");

async function handleClick(userId: string) {
  try {
    await DBOS.startWorkflow(processTask, {
      queueName: "task_queue",
      enqueueOptions: { deduplicationID: userId },
    })("task");
  } catch (e) {
    // DBOSQueueDuplicatedError - workflow already active for this user
    console.log("Task already in progress for user:", userId);
  }
}
```

Deduplication is per-queue. The deduplication ID is active while the workflow has status `DELAYED`, `ENQUEUED`, or `PENDING`. Once the workflow completes, it releases the deduplication ID and a new workflow with the same ID can be enqueued.

On a partitioned queue, deduplication IDs are unique across the whole queue, including all its partitions. To deduplicate within each partition separately, include the partition key in the deduplication ID.

### Singleton Workflows (return-existing)

If you want only one instance of a workflow to be active at a time and don't want to handle a thrown error, set `duplicationPolicy: 'return-existing'` on `DBOS.startWorkflow`. When a workflow with the same `deduplicationID` is already delayed, enqueued, or executing, this returns a handle to that existing workflow instead of throwing `DBOSQueueDuplicatedError`. The arguments passed by the colliding caller are discarded, and the returned handle resolves with the original workflow's result.

This requires both a `queueName` and `enqueueOptions.deduplicationID`.

```typescript
// Only one workflow with deduplicationID "singleton" can run on this queue
// at a time. Subsequent callers attach to it and receive its result.
const handle = await DBOS.startWorkflow(processTask, {
  queueName: "task_queue",
  enqueueOptions: { deduplicationID: "singleton" },
  duplicationPolicy: "return-existing",
})(task);
const result = await handle.getResult();
```

`duplicationPolicy` is also available on `client.enqueue`, but not on `DBOS.enqueueWorkflowWithOptions`; `client.enqueueInTransaction` throws on `'return-existing'`. The default `'reject'` throws `DBOSQueueDuplicatedError`.

Use cases for deduplication:
- Ensuring one active task per user
- Preventing duplicate form submissions
- Idempotent event processing
- Long-running singleton background jobs (`return-existing`)

Reference: [Deduplication](https://docs.dbos.dev/typescript/tutorials/queue-tutorial#deduplication)
