---
title: Partition Queues for Per-Entity Limits
impact: HIGH
impactDescription: Enables per-entity concurrency control with dynamic partitions
tags: queue, partition, per-user, dynamic, fairness
---

## Partition Queues for Per-Entity Limits

A queue is partitioned if you register it with any per-partition limit. Each partition key (e.g., a user ID) acts as a dynamically created "subqueue" with its own limits.

**Incorrect (global concurrency for per-user limits):**

```typescript
// globalConcurrency 1 blocks ALL users, not per-user
await DBOS.registerQueue("tasks", { globalConcurrency: 1 });
```

**Incorrect (removed in 5.0):**

```typescript
await DBOS.registerQueue("tasks", { partitionQueue: true, concurrency: 1 });
```

**Correct (per-partition limit):**

```typescript
await DBOS.registerQueue("tasks", { partitionConcurrency: 1 });

async function onUserTask(userID: string, task: string) {
  // At most 1 task per user, but different users run concurrently
  await DBOS.startWorkflow(processTask, {
    queueName: "tasks",
    enqueueOptions: { queuePartitionKey: userID },
  })(task);
}
```

Per-partition limits:
- `partitionConcurrency`: max workflows from one partition running at once across all processes
- `partitionWorkerConcurrency`: max workflows from one partition running at once on a single process
- `partitionRateLimit`: max workflows started from one partition per period (`{ limitPerPeriod, periodSec }`)

### Combining Queue-Wide and Per-Partition Limits

A partitioned queue enforces its per-partition limits **and** its queue-wide limits (`globalConcurrency`, `workerConcurrency`, `rateLimit`) at the same time:

```typescript
// Fair queue: at most 1 task per user, at most 10 tasks per process
await DBOS.registerQueue("fair_queue", { partitionConcurrency: 1, workerConcurrency: 10 });

// Mix and match freely
await DBOS.registerQueue("tenant_queue", {
  globalConcurrency: 100,
  workerConcurrency: 10,
  rateLimit: { limitPerPeriod: 1000, periodSec: 60 },
  partitionConcurrency: 25,
  partitionWorkerConcurrency: 2,
  partitionRateLimit: { limitPerPeriod: 50, periodSec: 60 },
});
```

Rules:
- Every enqueue on a partitioned queue must supply `queuePartitionKey`; a workflow enqueued without one stays `ENQUEUED` and is never dequeued
- Each per-partition concurrency limit must be `<=` its queue-wide counterpart, and `partitionWorkerConcurrency` must be `<=` `partitionConcurrency`
- Deduplication IDs are unique across the whole queue, including all partitions; include the partition key in the ID to deduplicate per partition

**Warning:** Setting any partition limit at runtime (e.g., `queue.setPartitionConcurrency(1)`) partitions the queue; clearing all of them unpartitions it. Workflows already enqueued without a partition key are not dequeued until the queue is unpartitioned.

Reference: [Partitioning Queues](https://docs.dbos.dev/typescript/tutorials/queue-tutorial#partitioning-queues)
