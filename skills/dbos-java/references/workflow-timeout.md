---
title: Use Durable Timeouts and Deadlines
impact: MEDIUM
impactDescription: Bounds workflow execution durably, surviving restarts
tags: workflow, timeout, deadline, cancellation
---

## Use Durable Timeouts and Deadlines

Set a timeout or deadline on a workflow with `StartWorkflowOptions`. Timeouts are durable — they are stored in the
database and survive restarts — and start-to-completion: an enqueued workflow's timeout does not start until it is
dequeued. When the timeout expires, the workflow and all of its children are cancelled at the beginning of the next
step.

**Incorrect (bounding a workflow from the caller's thread):**

```java
// A future timeout only abandons the caller — the workflow keeps running,
// and nothing survives a restart of this process.
var future = executor.submit(() -> proxy.longRunningWorkflow());
future.get(30, TimeUnit.MINUTES);
```

**Correct (durable timeout on the workflow itself):**

```java
import dev.dbos.transact.StartWorkflowOptions;
import dev.dbos.transact.workflow.Timeout;

// Relative timeout
var handle = dbos.startWorkflow(
    () -> proxy.longRunningWorkflow(),
    new StartWorkflowOptions().withTimeout(Duration.ofHours(12)));

// Absolute deadline
var deadlineHandle = dbos.startWorkflow(
    () -> proxy.longRunningWorkflow(),
    new StartWorkflowOptions().withDeadline(Instant.now().plus(Duration.ofHours(1))));

// Detach a child workflow from the parent's inherited timeout
dbos.startWorkflow(
    () -> self.childWorkflow(),
    new StartWorkflowOptions().withTimeout(Timeout.none()));
```

Rules and behavior:

- A timeout and a deadline cannot both be set on the same workflow
- Timeouts and deadlines propagate to child workflows; a child can override with its own `withTimeout(...)`,
  `withNoTimeout()`, or `Timeout.none()`
- `Timeout.of(Duration)` sets an explicit value, `Timeout.none()` opts out of any inherited timeout, and
  `Timeout.inherit()` is the default behavior
- Expiry sets the workflow's status to `CANCELLED`; a cancelled workflow can be restarted with `resumeWorkflow`
- For workflows invoked directly (not via `startWorkflow`), set the timeout on the calling context:

```java
try (var opts = new WorkflowOptions().withTimeout(Duration.ofMinutes(5)).setContext()) {
  proxy.workflow("input");
}
```

Reference: [Workflow Timeouts](https://docs.dbos.dev/java/tutorials/workflow-tutorial#workflow-timeouts)
