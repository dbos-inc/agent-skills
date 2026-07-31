---
title: Partition Queues for Per-Key Flow Control
impact: MEDIUM
impactDescription: Applies concurrency limits per tenant or user instead of globally
tags: queue, partitioning, multi-tenant, fairness, concurrency
---

## Partition Queues for Per-Key Flow Control

In a partitioned queue, every flow-control limit applies per partition key rather than to the queue as a whole.
Each key behaves like a dynamically created subqueue, which is how you enforce "one task at a time per user"
without creating a queue per user.

**Incorrect (a queue per tenant):**

```java
// Unbounded queue growth, and each queue must be registered before use
for (String tenantId : tenants) {
  dbos.registerQueue("tasks-" + tenantId, QueueOptions.setConcurrency(1));
}
```

**Correct (one partitioned queue keyed by tenant):**

```java
dbos.registerQueue("task-queue",
    QueueOptions.setConcurrency(1).andPartitionQueue(true));

void onUserTaskSubmission(String userId, Task task) {
  // Concurrency of 1 is enforced per partition key: at most one task per user
  // at a time, while different users run concurrently.
  var options = new StartWorkflowOptions()
      .withQueue("task-queue")
      .withQueuePartitionKey(userId);
  dbos.startWorkflow(() -> proxy.taskWorkflow(task), options);
}
```

Rules:

- A partition key is required when enqueueing to a partitioned queue, and rejected on a non-partitioned queue
- Partition keys and deduplication IDs cannot be used together
- Concurrency and rate limits apply per partition, so a global cap needs a second, non-partitioned queue

To enforce both per-key and global limits, chain two queues: enqueue a "concurrency manager" workflow on the
partitioned queue, and have it enqueue the real workflow on a non-partitioned queue and await the result.

```java
dbos.registerQueue("concurrency-queue", QueueOptions.setWorkerConcurrency(5));
dbos.registerQueue("partitioned-queue",
    QueueOptions.setConcurrency(1).andPartitionQueue(true));

@Workflow
public void onUserTaskSubmission(String userId, Task task) {
  dbos.startWorkflow(() -> self.concurrencyManager(task),
      new StartWorkflowOptions().withQueue("partitioned-queue").withQueuePartitionKey(userId));
}

@Workflow
public String concurrencyManager(Task task) throws Exception {
  var handle = dbos.startWorkflow(() -> self.processTask(task),
      new StartWorkflowOptions().withQueue("concurrency-queue"));
  return handle.getResult();
}
```

Reference: [Partitioning Queues](https://docs.dbos.dev/java/tutorials/queue-tutorial#partitioning-queues)
