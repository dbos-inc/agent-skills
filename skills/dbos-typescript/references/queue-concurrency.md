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
await DBOS.registerQueue("limited_tasks", { globalConcurrency: 10 });
```

**In-order processing (sequential):**

```typescript
async function processEventFn(event: string) {
  // ...
}
const processEvent = DBOS.registerWorkflow(processEventFn);

// After DBOS.launch(): only one task at a time - guarantees order
await DBOS.registerQueue("sequential_queue", { globalConcurrency: 1 });

app.post("/events", async (req, res) => {
  await DBOS.startWorkflow(processEvent, {
    queueName: "sequential_queue",
  })(req.body.event);
  res.send("Queued!");
});
```

Worker concurrency is recommended for most use cases. Take care with global concurrency as any `PENDING` workflow on the queue counts toward the limit, including workflows from previous application versions.

If both are set, `workerConcurrency` must be less than or equal to `globalConcurrency`. The `concurrency` option is a deprecated alias for `globalConcurrency`.

To change limits at runtime without redeploying, see `queue-management.md`.

Reference: [Managing Concurrency](https://docs.dbos.dev/typescript/tutorials/queue-tutorial#managing-concurrency)
