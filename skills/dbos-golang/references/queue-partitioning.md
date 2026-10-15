---
title: Partition Queues for Per-Entity Limits
impact: HIGH
impactDescription: Enables per-entity concurrency control with dynamic partitions
tags: queue, partition, per-user, dynamic, fairness
---

## Partition Queues for Per-Entity Limits

A queue is partitioned if you register it with any per-partition limit. Each partition key (e.g., a user ID) acts as a dynamically created "subqueue" with its own limits.

**Incorrect (global concurrency for per-user limits):**

```go
// Global concurrency=1 blocks ALL users, not per-user
queue, err := dbos.RegisterQueue(ctx, "tasks",
	dbos.WithGlobalConcurrency(1),
)
```

**Incorrect (deprecated `WithPartitionQueue`, which applies the queue-wide limits to each partition and cannot combine the two):**

```go
queue, err := dbos.RegisterQueue(ctx, "tasks",
	dbos.WithPartitionQueue(),
	dbos.WithGlobalConcurrency(1),
)
```

**Correct (per-partition limit):**

```go
queue, err := dbos.RegisterQueue(ctx, "tasks",
	dbos.WithPartitionConcurrency(1),
)

func onUserTask(ctx dbos.Context, userID, task string) error {
	// At most 1 task per user, but different users run concurrently
	_, err := dbos.RunWorkflow(ctx, processTask, task,
		dbos.WithQueue(queue),
		dbos.WithQueuePartitionKey(userID),
	)
	return err
}
```

Per-partition limits (v1.3.0+):
- `WithPartitionConcurrency(n)`: max workflows from one partition running at once across all processes
- `WithPartitionWorkerConcurrency(n)`: max workflows from one partition running at once on a single process
- `WithPartitionRateLimiter(&dbos.RateLimiter{Limit, Period})`: max workflows started from one partition per period

### Combining queue-wide and per-partition limits

A partitioned queue enforces its per-partition limits **and** its queue-wide limits (`WithGlobalConcurrency`, `WithWorkerConcurrency`, `WithRateLimiter`) at the same time:

```go
// Fair queue: at most 1 task per user, at most 10 tasks per process
fairQueue, err := dbos.RegisterQueue(ctx, "fair_queue",
	dbos.WithPartitionConcurrency(1),
	dbos.WithWorkerConcurrency(10),
)

// Mix and match freely
tenantQueue, err := dbos.RegisterQueue(ctx, "tenant_queue",
	dbos.WithGlobalConcurrency(100),
	dbos.WithWorkerConcurrency(10),
	dbos.WithRateLimiter(&dbos.RateLimiter{Limit: 1000, Period: 60 * time.Second}),
	dbos.WithPartitionConcurrency(25),
	dbos.WithPartitionWorkerConcurrency(2),
	dbos.WithPartitionRateLimiter(&dbos.RateLimiter{Limit: 50, Period: 60 * time.Second}),
)
```

Rules:
- Every enqueue on a partitioned queue must supply `WithQueuePartitionKey` (`WithEnqueueQueuePartitionKey` from a client); a workflow enqueued without one stays `ENQUEUED` and is never dequeued
- Partition keys cannot be used with non-partitioned queues, nor together with deduplication IDs
- Each per-partition concurrency limit must be `<=` its queue-wide counterpart, and the partition worker concurrency must be `<=` the partition concurrency; `RegisterQueue` returns an error otherwise
- Limits can be changed at runtime with the `Queue` setters (`SetPartitionConcurrency`, `SetPartitionWorkerConcurrency`, `SetPartitionRateLimit`; pass `nil` to clear). Setting any partition limit partitions the queue: workflows already enqueued without a partition key are not dequeued until the queue is unpartitioned again

Reference: [Partitioning Queues](https://docs.dbos.dev/golang/tutorials/queue-tutorial#partitioning-queues)
