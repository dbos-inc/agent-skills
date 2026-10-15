---
title: Set Queue Priority for Workflows
impact: HIGH
impactDescription: Prioritizes important workflows over lower-priority ones
tags: queue, priority, ordering, importance
---

## Set Queue Priority for Workflows

Set a priority when enqueuing a workflow to have it dequeued before lower-priority work. Lower numbers indicate higher priority. Every queue dequeues in priority order; there is nothing to enable on the queue.

**Incorrect (urgent work waits behind the backlog):**

```go
// FIFO only: an urgent task enqueued behind thousands of background tasks runs last
dbos.RunWorkflow(ctx, processTask, "urgent-task",
	dbos.WithQueue(queue),
)
```

**Correct (prioritized enqueue):**

```go
queue, err := dbos.RegisterQueue(ctx, "tasks")

// High priority task (lower number = higher priority)
dbos.RunWorkflow(ctx, processTask, "urgent-task",
	dbos.WithQueue(queue),
	dbos.WithPriority(1),
)

// Low priority task
dbos.RunWorkflow(ctx, processTask, "background-task",
	dbos.WithQueue(queue),
	dbos.WithPriority(100),
)
```

Priority rules:
- Range: `1` to `2,147,483,647`
- Lower number = higher priority
- Workflows **without** assigned priorities have the highest priority (run first)
- Workflows with the same priority are dequeued in FIFO order
- From an external application, use `WithEnqueuePriority` with `dbos.Enqueue` ([client-enqueue.md](client-enqueue.md))

Reference: [Priority](https://docs.dbos.dev/golang/tutorials/queue-tutorial#priority)
