---
title: Delay Enqueued Workflows
impact: LOW-MEDIUM
impactDescription: Schedules future work durably without holding a worker
tags: queue, delay, scheduling, DELAYED, backoff
---

## Delay Enqueued Workflows

`withDelay` postpones when an enqueued workflow becomes eligible to start. The workflow sits in `DELAYED` state and
becomes `ENQUEUED` when the delay elapses, so no worker or thread is occupied while waiting.

**Incorrect (sleeping to postpone the start):**

```java
// Occupies a worker slot for the whole wait and blocks other queued work
@Workflow
public void reminder(String userId) throws Exception {
  dbos.sleep(Duration.ofHours(24));
  dbos.runStep(() -> sendReminder(userId), "sendReminder");
}
```

**Correct (delay at enqueue time):**

```java
var options = new StartWorkflowOptions()
    .withQueue("reminder-queue")
    .withDelay(Duration.ofHours(24));

dbos.startWorkflow(() -> proxy.reminder(userId), options);
```

You can also delay a workflow that is already enqueued or pending:

```java
// Push it back by a relative duration
dbos.setWorkflowDelay(workflowId, Duration.ofMinutes(30));

// Or until an absolute time
dbos.setWorkflowDelay(workflowId, Instant.now().plus(Duration.ofHours(2)));
```

Notes:

- `withDelay` only applies when enqueueing, and the duration must be positive
- The delay is measured from enqueue time; timeouts still start only when the workflow begins executing
- `DELAYED` workflows count against a deduplication ID, so a delayed retry blocks a duplicate enqueue for the
  same key
- For long waits inside a running workflow, use durable sleep instead ([pattern-sleep.md](pattern-sleep.md))

Reference: [Delaying Execution](https://docs.dbos.dev/java/tutorials/queue-tutorial#delaying-execution)
