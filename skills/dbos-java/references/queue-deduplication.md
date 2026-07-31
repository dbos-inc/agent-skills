---
title: Deduplicate Enqueued Workflows
impact: MEDIUM
impactDescription: Guarantees at most one active workflow per key on a queue
tags: queue, deduplication, idempotency, enqueue
---

## Deduplicate Enqueued Workflows

A deduplication ID ensures that at most one workflow with that ID is enqueued or executing on a given queue at any
time. Use it when repeated triggers for the same entity should collapse into a single active run — one sync per
user, one rebuild per project.

**Incorrect (checking for a running workflow first):**

```java
// Racy: two threads can both see "nothing running" and enqueue duplicates
if (dbos.listWorkflows(new ListWorkflowsInput()
        .withStatus(WorkflowState.PENDING)
        .withWorkflowIdPrefix("sync-" + userId)).isEmpty()) {
  dbos.startWorkflow(() -> proxy.syncUser(userId),
      new StartWorkflowOptions().withQueue("sync-queue"));
}
```

**Correct (deduplication ID enforced by the database):**

```java
import dev.dbos.transact.exceptions.DBOSQueueDuplicatedException;

var options = new StartWorkflowOptions()
    .withQueue("sync-queue")
    .withDeduplicationId(userId);

try {
  var handle = dbos.startWorkflow(() -> proxy.syncUser(userId), options);
  return handle.workflowId();
} catch (DBOSQueueDuplicatedException e) {
  // A sync for this user is already ENQUEUED, DELAYED, or PENDING
  return null;
}
```

Behavior:

- Deduplication is scoped to one queue and one ID; the same ID may be active on a different queue
- The ID is held while the workflow is `ENQUEUED`, `DELAYED`, or `PENDING`, and released when it completes,
  fails, or is cancelled — the next call then enqueues a fresh workflow
- Enqueueing a duplicate throws, so callers must handle `DBOSQueueDuplicatedException`
- Deduplication IDs cannot be combined with queue partition keys
- To collapse a burst of calls into one delayed execution instead of rejecting them, use the debouncer
  ([pattern-debouncing.md](pattern-debouncing.md)); to make repeated calls resolve to the same execution, set an
  explicit workflow ID ([pattern-idempotency.md](pattern-idempotency.md))

Reference: [Deduplication](https://docs.dbos.dev/java/tutorials/queue-tutorial#deduplication)
