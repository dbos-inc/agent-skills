---
title: Cancel, Resume, and Fork Workflows
impact: MEDIUM
impactDescription: Enables operational control over long-running workflows
tags: workflow, cancel, resume, fork, management
---

## Cancel, Resume, and Fork Workflows

DBOS provides functions to cancel, resume, and fork workflows for operational control.

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

Cancellation sets the workflow status to `CANCELLED` and preempts execution at the beginning of the next step. Cancelling also cancels all child workflows.

Resume restarts a workflow from its last completed step. Use this for workflows that are cancelled or have exceeded their maximum recovery attempts. You can also use this to start an enqueued workflow immediately, bypassing its queue.

Resume onto a different queue with `WithResumeQueue`:

```go
handle, err := dbos.ResumeWorkflow[string](ctx, workflowID,
    dbos.WithResumeQueue("priority"))
```

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
    StartStep:          2,             // Fork from step 2
    ForkedWorkflowID:   "new-wf-id",   // Optional
    ApplicationVersion: "2.0.0",       // Optional
    QueueName:          "priority",    // Optional: enqueue the fork on a specific queue
    QueuePartitionKey:  "user-123",    // Optional: partition key (requires QueueName)
})
result, err := forkHandle.GetResult()
```

Forking creates a new workflow with a new ID, copying the original workflow's inputs and step outputs up to the selected step. `QueuePartitionKey` requires `QueueName` to be set.

### Delay a queued workflow

Push back the dequeue time of a queued workflow with `SetWorkflowDelay`. Provide exactly one of `WithDelayDuration` (relative) or `WithDelayUntil` (absolute). See [pattern-delayed-execution.md](pattern-delayed-execution.md).

Reference: [Workflow Management](https://docs.dbos.dev/golang/tutorials/workflow-management)
