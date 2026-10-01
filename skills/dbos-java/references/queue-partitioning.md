---
title: Partition Queues for Per-Key Flow Control
impact: MEDIUM
impactDescription: Applies concurrency limits per tenant or user alongside queue-wide limits
tags: queue, partitioning, multi-tenant, fairness, concurrency
---

## Partition Queues for Per-Key Flow Control

A partitioned queue enforces limits per partition key. Each key behaves like a dynamically created subqueue, which
is how you enforce "one task at a time per user" without creating a queue per user.

Setting any per-partition limit partitions the queue: `withPartitionConcurrency`, `withPartitionWorkerConcurrency`
or `withPartitionRateLimit`.

**Incorrect (a queue per tenant):**

```java
// Unbounded queue growth, and each queue must be registered before use
for (String tenantId : tenants) {
  dbos.registerQueue("tasks-" + tenantId, new QueueOptions().withConcurrency(1));
}
```

**Correct (one partitioned queue keyed by tenant):**

```java
dbos.launch();
dbos.registerQueue("task-queue", new QueueOptions().withPartitionConcurrency(1));

void onUserTaskSubmission(String userId, Task task) {
  // At most one task per user at a time, while different users run concurrently.
  var options = new StartWorkflowOptions()
      .withQueue("task-queue")
      .withQueuePartitionKey(userId);
  dbos.startWorkflow(() -> proxy.taskWorkflow(task), options);
}
```

### Per-key and queue-wide limits on one queue

Queue-wide limits mean queue-wide, and per-partition limits mean per key. Set both to bound a shared resource
while keeping one tenant from monopolizing it:

```java
// At most 10 running across the deployment, and at most 1 per user
dbos.registerQueue("task-queue",
    new QueueOptions().withConcurrency(10).withPartitionConcurrency(1));

// Every limit has a per-partition counterpart
dbos.registerQueue("api-queue",
    new QueueOptions()
        .withConcurrency(20)
        .withWorkerConcurrency(5)
        .withRateLimit(100, Duration.ofSeconds(60))
        .withPartitionConcurrency(2)
        .withPartitionWorkerConcurrency(1)
        .withPartitionRateLimit(10, Duration.ofSeconds(60)));
```

A limit enforced at a narrower scope may never exceed one enforced at a wider scope. Limits are compared only when
both are set. Registering or updating a queue fails if:

- any concurrency limit, rate-limit max or rate-limit period is zero or negative
- `partitionConcurrency` exceeds `concurrency`
- `partitionWorkerConcurrency` exceeds `partitionConcurrency`, `workerConcurrency` or `concurrency`
- `workerConcurrency` exceeds `concurrency`

### Rules

- Per-partition limits are supported only on database-backed queues — `registerQueue(String, QueueOptions)` after
  launch. In-memory `registerQueue(Queue)` rejects them, because an in-memory queue is never written and its limits
  could not be shared with other processes
- A partition key is required when enqueueing to a partitioned queue, and rejected on a non-partitioned queue
- Partition keys and deduplication IDs cannot be used together
- With many partition keys, prefer `partitionConcurrency` 1 and no queue-wide concurrency, rate limit or partition
  rate limit (worker limits are fine); since 1.2 that shape dequeues fastest
- Partitioning an existing queue strands whatever is already enqueued on it: those rows have no partition key, and
  a partitioned queue dequeues only from the keys present. Drain it first; to rescue stranded workflows, move them
  to a queue that is not partitioned with `dbos.resumeWorkflow(workflowId, queueName)`

### The deprecated partitionQueue flag

`andPartitionQueue(true)` is deprecated for removal since 1.1. Used alone, it enforces the *queue-wide* limits per
partition — `withConcurrency(1).andPartitionQueue(true)` means one per key, not one per queue. Combined with any
per-partition option at registration it is accepted but does nothing: the queue is partitioned by its per-partition
limits and `concurrency` goes back to meaning queue-wide, so
`withConcurrency(5).withPartitionConcurrency(2).andPartitionQueue(true)` runs at most five tasks across the whole queue,
not five per key. An `updateQueue` that sets the flag on a queue partitioned by its limits throws. Do not mix them. A
queue registered with the flag alone has its limits frozen: `updateQueue` throws `IllegalArgumentException` for any
limit change, queue-wide or per-partition (only `pollingInterval` can still change), because the two modes disagree
about what `concurrency` means. Re-register the queue with per-partition limits instead.

```java
// Deprecated: concurrency is enforced per key
dbos.registerQueue("task-queue", new QueueOptions().withConcurrency(1).andPartitionQueue(true));

// Current equivalent
dbos.registerQueue("task-queue", new QueueOptions().withPartitionConcurrency(1));
```

Reference: [Partitioning Queues](https://docs.dbos.dev/java/tutorials/queue-tutorial#partitioning-queues)
