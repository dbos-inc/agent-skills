---
title: Prioritize Workflows on a Queue
impact: MEDIUM
impactDescription: Ensures urgent work is dequeued before routine work
tags: queue, priority, ordering, fifo
---

## Prioritize Workflows on a Queue

Set a priority on enqueued workflows to dequeue urgent work first. Every queue dequeues in priority order; no queue
configuration is needed. Priority values range from 0 to 2,147,483,647 where a *lower* number means higher
priority; workflows with the same priority keep FIFO order.

**Incorrect (separate queues to fake priority):**

```java
// Two queues cannot express relative ordering: a worker polling both
// has no way to know that urgent work should win
dbos.registerQueue("urgent-queue", QueueOptions.empty());
dbos.registerQueue("normal-queue", QueueOptions.empty());
```

**Correct (one queue, priority set per workflow):**

```java
dbos.registerQueue("task-queue", QueueOptions.empty());

// Higher priority (dequeued first)
dbos.startWorkflow(() -> proxy.processTask(urgentTask),
    new StartWorkflowOptions().withQueue("task-queue").withPriority(1));

// Lower priority
dbos.startWorkflow(() -> proxy.processTask(bulkTask),
    new StartWorkflowOptions().withQueue("task-queue").withPriority(100));
```

Behavior:

- `setPriorityEnabled` is deprecated for removal since 1.1 and ignored: every queue already dequeues in priority order
- A negative priority throws `IllegalArgumentException` as soon as it is set (`withPriority` on `StartWorkflowOptions`
  or `EnqueueOptions`), or from `debounce()` for a debouncer
- Workflows enqueued *without* a priority get priority 0, the highest, so they outrank every prioritized workflow —
  either assign priorities consistently on a queue or not at all
- Priority affects dequeue order only; it does not preempt workflows that are already running
- Priority composes with concurrency and rate limits, which still cap how much runs at once

Reference: [Priority](https://docs.dbos.dev/java/tutorials/queue-tutorial#priority)
