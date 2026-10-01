---
title: Use Durable Workflow Timeouts
impact: MEDIUM
impactDescription: Bounds workflow execution durably, surviving restarts
tags: workflow, timeout, deadline, cancellation
---

## Use Durable Workflow Timeouts

Set a timeout on a workflow with `StartWorkflowOptions` (or `EnqueueOptions`). Timeouts are durable — they are
stored in the database and survive restarts — and start-to-completion: an enqueued workflow's timeout does not start
until it is dequeued. When the workflow starts, its timeout becomes an absolute deadline. When that deadline passes,
the workflow and all of its children are cancelled at the beginning of the next step.

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

var handle = dbos.startWorkflow(
    () -> proxy.longRunningWorkflow(),
    new StartWorkflowOptions().withTimeout(Duration.ofHours(12)));

// Inside a workflow: detach a child from the parent's deadline
dbos.startWorkflow(
    () -> self.childWorkflow(),
    new StartWorkflowOptions().withTimeout(Timeout.none()));
```

**Incorrect (absolute deadline):**

```java
// Deprecated for removal since 1.2: no other DBOS SDK lets a caller set a deadline
new StartWorkflowOptions().withDeadline(Instant.now().plus(Duration.ofHours(1)));
```

`withDeadline` and `deadline()` on `StartWorkflowOptions`, `EnqueueOptions` and `WorkflowOptions` are deprecated for
removal — do not use them. Use a timeout; for a workflow that starts at once,
`withTimeout(Duration.between(Instant.now(), deadline))` is the same bound. Building a `StartWorkflowOptions` or
`EnqueueOptions` with both an explicit timeout and a deadline throws `IllegalArgumentException`.

Child workflows (1.2+): a child with no bound of its own inherits the parent's *deadline*, not its timeout, so a
child — even a queued one — can never outlive its parent. A queued child dequeued after the inherited deadline has
passed is cancelled. A child's bound comes from the first of these that is set:

1. the call's own options (`StartWorkflowOptions` / `EnqueueOptions`): a timeout, `withNoTimeout()` /
   `Timeout.none()`, or `Timeout.inherit()`
2. an enclosing `WorkflowOptions` block
3. the parent's deadline

Rules and behavior:

- A bound given for the call replaces the whole bound set by `WorkflowOptions`; an inner `WorkflowOptions` block that
  sets a timeout or `withNoTimeout()` replaces the outer block's bound until it closes
- `Timeout.of(Duration)` sets an explicit value, `Timeout.none()` runs with no timeout and does not inherit the
  parent's deadline, and `Timeout.inherit()` takes the parent's deadline even inside a `WorkflowOptions` block that
  sets a bound — outside a workflow, it means no timeout
- A child that inherited its bound has a null timeout (`WorkflowStatus.timeout()`, `DBOSContext.getTimeout()`) but
  a `deadline()`. Resuming it clears that deadline and leaves it unbounded
- Expiry sets the workflow's status to `CANCELLED`; a cancelled workflow can be restarted with `resumeWorkflow`
- A debounced workflow never inherits a timeout or deadline; use `Debouncer.withTimeout`
  ([pattern-debouncing.md](pattern-debouncing.md))
- For workflows invoked directly (not via `startWorkflow`), set the timeout on the calling context:

```java
try (var opts = new WorkflowOptions().withTimeout(Duration.ofMinutes(5)).setContext()) {
  proxy.workflow("input");
}
```

Reference: [Workflow Timeouts](https://docs.dbos.dev/java/tutorials/workflow-tutorial#workflow-timeouts)
