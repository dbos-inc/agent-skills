---
title: Control Queue Concurrency
impact: HIGH
impactDescription: Prevents resource exhaustion with concurrent limits
tags: queue, concurrency, workerConcurrency, limits
---

## Control Queue Concurrency

Queues support worker-level and global concurrency limits to prevent resource exhaustion.

**Incorrect (no concurrency control):**

```typescript
await DBOS.registerQueue("heavy_tasks"); // No limits - could exhaust memory
```

**Correct (worker concurrency):**

```typescript
// Each process runs at most 5 tasks from this queue
await DBOS.registerQueue("heavy_tasks", { workerConcurrency: 5 });
```

**Correct (global concurrency):**

```typescript
// At most 10 tasks run across ALL processes
await DBOS.registerQueue("limited_tasks", { concurrency: 10 });
```

**In-order processing (sequential):**

```typescript
// Only one task at a time - guarantees order
await DBOS.registerQueue("sequential_queue", { concurrency: 1 });

async function processEventFn(event: string) {
  // ...
}
const processEvent = DBOS.registerWorkflow(processEventFn);

app.post("/events", async (req, res) => {
  await DBOS.startWorkflow(processEvent, {
    queueName: "sequential_queue",
  })(req.body.event);
  res.send("Queued!");
});
```

Worker concurrency is recommended for most use cases. Take care with global concurrency as any `PENDING` workflow on the queue counts toward the limit, including workflows from previous application versions.

When using worker concurrency, each process must have a unique `executorID` set in configuration (this is automatic with DBOS Conductor or Cloud).

### Reconfiguring at Runtime

Because queue configuration lives in the system database, you can change a queue's concurrency at runtime without redeploying. Workers pick up the new configuration on their next polling iteration.

```typescript
const queue = await DBOS.retrieveQueue("heavy_tasks");
if (queue !== null) {
  await queue.setConcurrency(20);
  await queue.setWorkerConcurrency(2);
}
```

If your application also calls `DBOS.registerQueue` on startup, the next process to launch can overwrite your runtime changes. Either update the `registerQueue` call to match, or pass `onConflict: 'never_update'` to preserve runtime values.

Reference: [Managing Concurrency](https://docs.dbos.dev/typescript/tutorials/queue-tutorial#managing-concurrency)
