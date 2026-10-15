---
title: Cancel, Resume, Fork, and Rewind Workflows
impact: MEDIUM
impactDescription: Enables operational control over long-running workflows
tags: workflow, cancel, resume, fork, rewind, management
---

## Cancel, Resume, Fork, and Rewind Workflows

DBOS provides functions to cancel, resume, fork, and rewind workflows for operational control.

**Incorrect (no way to handle stuck or failed workflows):**

```go
// Workflow is stuck or failed - no recovery mechanism
handle, _ := dbos.RunWorkflow(ctx, processTask, "data")
// If the workflow fails, there's no way to retry or recover
```

**Correct (using cancel, resume, and fork):**

```go
// Cancel a workflow - stops at its next step
err := dbos.CancelWorkflow(ctx, workflowID)

// Resume from the last completed step
handle, err := dbos.ResumeWorkflow[string](ctx, workflowID)
result, err := handle.GetResult()
```

Cancellation sets the workflow status to `CANCELLED` and preempts execution at the beginning of the next step. Pass `dbos.WithCancelChildren()` to also cancel all child workflows.

Resume restarts a workflow from its last completed step. Use this for workflows that are cancelled or have exceeded their maximum recovery attempts. You can also use this to start an enqueued workflow immediately, bypassing its queue.

Resume onto a different queue with `WithResumeQueue`:

```go
handle, err := dbos.ResumeWorkflow[string](ctx, workflowID,
    dbos.WithResumeQueue("priority"))
```

Resuming, like cancelling, rewinding, or recovering a workflow, takes ownership of it away from any execution still running it: every execution holds a fencing token, and a stale one can no longer checkpoint over the current owner. Its next DBOS operation returns an error matching `dbos.ErrConflictingWorkflowID`; return that error from the workflow function (never swallow it) so the execution parks until the owner settles and adopts its outcome.

### Bulk operations

`CancelWorkflows` and `ResumeWorkflows` operate on many IDs in a single DB round-trip. Unlike their single-ID variants, missing or terminal workflows are silently skipped instead of erroring:

```go
err := dbos.CancelWorkflows(ctx, []string{"wf-1", "wf-2", "wf-3"})

handles, err := dbos.ResumeWorkflows[string](ctx,
    []string{"wf-1", "wf-2"},
    dbos.WithResumeQueue("priority"))
```

### Fork

Fork a workflow from a specific step:

```go
// List steps to find the right step ID
steps, err := dbos.GetWorkflowSteps(ctx, workflowID)

// Fork from a specific step
forkHandle, err := dbos.ForkWorkflow[string](ctx, dbos.ForkWorkflowInput{
    OriginalWorkflowID: workflowID,
    StartStep:          2,               // Fork from step 2
    ForkedWorkflowID:   "new-wf-id",     // Optional
    ApplicationVersion: "2.0.0",         // Optional
    QueueName:          "priority",      // Optional: enqueue the fork on a specific queue
    QueuePartitionKey:  "user-123",      // Optional: partition key (requires QueueName)
    Timeout:            30 * time.Minute, // Optional: the original's timeout is not inherited
})
result, err := forkHandle.GetResult()
```

Forking creates a new workflow with a new ID, copying the original workflow's inputs and step outputs up to the selected step. `QueuePartitionKey` requires `QueueName` to be set. `Timeout` starts when the fork begins executing (at dequeue, if enqueued).

To fork a parent whose children were also forked, map the original child IDs to the new ones with `ReplacementChildren`, so the copied child-start steps point at the new children:

```go
childHandle, err := dbos.ForkWorkflow[ChildResult](ctx, dbos.ForkWorkflowInput{
    OriginalWorkflowID: "old-child-id",
    ForkedWorkflowID:   "new-child-id",
    StartStep:          2,
})
parentHandle, err := dbos.ForkWorkflow[ParentResult](ctx, dbos.ForkWorkflowInput{
    OriginalWorkflowID:  "parent-workflow-id",
    StartStep:           6,
    ReplacementChildren: map[string]string{"old-child-id": "new-child-id"},
})
```

### Rewind

`dbos.RewindWorkflow` re-executes a workflow from a step **in place, keeping its workflow ID** (v1.5.0+). Use it when other code refers to the workflow by ID (an idempotency key derived from an order ID, senders, event or stream readers): they keep working, and child workflows keep their IDs.

**Incorrect (forking when callers depend on the original ID):**

```go
// The order's workflow ID is "order-123"; the fork gets a new ID,
// so webhooks sending to "order-123" never reach the new execution.
dbos.ForkWorkflow[string](ctx, dbos.ForkWorkflowInput{OriginalWorkflowID: "order-123", StartStep: 3})
```

**Correct (rewind in place):**

```go
// Only a terminal workflow (SUCCESS, ERROR, CANCELLED, MAX_RECOVERY_ATTEMPTS_EXCEEDED)
// can be rewound; cancel a running workflow first.
handle, err := dbos.RewindWorkflow[string](ctx, "order-123", dbos.WithRewindStartStep(3))
result, err := handle.GetResult()

// Omit the start step to re-run from the beginning; optionally move onto fixed code
handle, err = dbos.RewindWorkflow[string](ctx, "order-123", dbos.WithRewindApplicationVersion("2.0.0"))
```

Options: `WithRewindStartStep(step)` (default 0; must match a `StepID` from `GetWorkflowSteps`), `WithRewindApplicationVersion(version)`, `WithRewindQueue(name)` and `WithRewindQueuePartitionKey(key)` (defaults to an internal queue that dequeues immediately), `WithRewindDataSources(ds...)`.

Rewind rules:
- Steps with ID `>=` the start step are discarded and re-executed; earlier steps replay their recorded outputs
- Clears the output or error, restores events set at or after the start step to their earlier values, and deletes messages consumed at or after the start step as well as every unconsumed message (send new ones only once the rewind has returned)
- Streams are append-only: new values are appended, and streams closed at or after the start step are reopened
- Child workflows are not modified, even those started after the start step; a child that already succeeded is adopted by the replaying parent. To re-run a child, delete or rewind it separately before rewinding the parent
- On a `Context`, the datasource transaction checkpoints of the rewound steps are deleted, so the replay re-runs those transactions ([step-transactions.md](step-transactions.md)). A standalone client has no datasource registry: pass the datasources with `WithRewindDataSources`, or rewound transactions are not re-executed

### Delay a queued workflow

Push back the dequeue time of a queued workflow with `SetWorkflowDelay`. Provide exactly one of `WithDelayDuration` (relative) or `WithDelayUntil` (absolute). See [pattern-delayed-execution.md](pattern-delayed-execution.md).

Reference: [Workflow Management](https://docs.dbos.dev/golang/tutorials/workflow-management)
