---
title: Delay Workflow Execution
impact: MEDIUM
impactDescription: Schedule one-off work for later without polling or external timers
tags: pattern, delay, scheduled, queue
---

## Delay Workflow Execution

Use a delay when you need a workflow to run later (a retry timer, a TTL, a scheduled email) and don't want to write an external scheduler. The workflow is recorded immediately with status `ENQUEUED` and `delay_until` populated; the queue runner only dequeues it once that time passes.

This is *not* the same as a recurring cron schedule. For cron see [pattern-scheduled.md](pattern-scheduled.md). For inline durable waits inside a running workflow, see [pattern-sleep.md](pattern-sleep.md).

### Enqueueing with a delay

**Incorrect (sleeping in a goroutine to delay a workflow):**

```go
// Lost if the process crashes; no durability, no introspection
go func() {
    time.Sleep(24 * time.Hour)
    dbos.RunWorkflow(ctx, sendReminder, userID)
}()
```

**Correct (WithDelay on enqueue):**

```go
// Relative delay on a queued workflow
handle, _ := dbos.RunWorkflow(ctx, sendReminder, userID,
    dbos.WithQueue("notifications"),
    dbos.WithDelay(24*time.Hour))
```

From the client (external app):

```go
handle, _ := client.Enqueue("notifications", "sendReminder", userID,
    dbos.WithEnqueueDelay(24*time.Hour))
```

### Updating the delay later

`SetWorkflowDelay` re-targets a workflow whose status is `DELAYED`. Provide exactly one of `WithDelayDuration` (relative to now) or `WithDelayUntil` (absolute):

```go
// Push it back another hour
dbos.SetWorkflowDelay(ctx, workflowID,
    dbos.WithDelayDuration(time.Hour))

// Or set an absolute fire time
dbos.SetWorkflowDelay(ctx, workflowID,
    dbos.WithDelayUntil(time.Date(2026, 6, 1, 9, 0, 0, 0, time.UTC)))
```

Calling with both options or neither returns an error. The same option is available from `Client.SetWorkflowDelay`.

### Cancelling a delayed workflow

`CancelWorkflow` works on delayed workflows the same as any other status. To run it immediately instead of waiting, use `ResumeWorkflow` — it bypasses the delay and re-enqueues the workflow.

Reference: [Queues](https://docs.dbos.dev/golang/tutorials/queue-tutorial)
