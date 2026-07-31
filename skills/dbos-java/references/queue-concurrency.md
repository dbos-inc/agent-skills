---
title: Limit Queue Concurrency to Protect Resources
impact: HIGH
impactDescription: Prevents resource exhaustion from too many concurrent workflows
tags: queue, concurrency, worker, limits, resources
---

## Limit Queue Concurrency to Protect Resources

Concurrency limits cap how many workflows from a queue run at once. Worker concurrency applies per DBOS process and
is the recommended control; global concurrency applies across every process sharing the system database.

**Incorrect (unlimited concurrency for memory-hungry work):**

```java
// Every enqueued workflow starts as soon as a worker polls: memory blows up
dbos.registerQueue("ml-queue", QueueOptions.empty());
```

**Correct (bounded per-process concurrency):**

```java
// Each process runs at most 5 of these workflows at a time
dbos.registerQueue("ml-queue", QueueOptions.setWorkerConcurrency(5));

// Cap across the whole deployment (use sparingly, see caveat below)
dbos.registerQueue("api-queue", QueueOptions.setConcurrency(10));

// Both together: at most 10 globally, at most 2 per process
dbos.registerQueue("mixed-queue",
    QueueOptions.setConcurrency(10).andWorkerConcurrency(2));
```

Choosing a limit:

- `workerConcurrency` — maximum concurrent workflows from this queue in a single process. Use it for CPU- or
  memory-intensive work so no process is overwhelmed; total throughput scales with the number of processes.
- `concurrency` — maximum concurrent workflows from this queue across all processes. Use it to protect a shared
  downstream resource such as a database connection pool or a licensed service.

Caveat for global concurrency: every `PENDING` workflow on the queue counts toward the limit, including workflows
left behind by earlier application versions. A stuck workflow therefore consumes a slot until it is cancelled or
resumed.

To rate-limit starts rather than cap in-flight work, use a rate limit
([queue-rate-limiting.md](queue-rate-limiting.md)). To apply limits per tenant or user, use a partitioned queue
([queue-partitioning.md](queue-partitioning.md)).

Reference: [Managing Concurrency](https://docs.dbos.dev/java/tutorials/queue-tutorial#managing-concurrency)
