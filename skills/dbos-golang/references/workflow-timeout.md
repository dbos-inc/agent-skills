---
title: Set Workflow Timeouts
impact: CRITICAL
impactDescription: Prevents workflows from running indefinitely
tags: workflow, timeout, cancellation, duration
---

## Set Workflow Timeouts

Set a timeout for a workflow by deriving a deadline context with `dbos.WithTimeout`. When the timeout expires, the workflow and all its children are cancelled.

**Incorrect (no timeout for potentially long workflow):**

```go
// No timeout - could run indefinitely
handle, err := dbos.RunWorkflow(ctx, processTask, "data")
```

**Correct (with timeout):**

```go
// Create a context with a 5-minute timeout
timedCtx, cancel := dbos.WithTimeout(ctx, 5*time.Minute)
defer cancel()

handle, err := dbos.RunWorkflow(timedCtx, processTask, "data")
if err != nil {
	log.Fatal(err)
}
```

Key timeout behaviors:
- Timeouts are **start-to-completion**: the timeout begins when the workflow starts execution, not when it's enqueued
- Timeouts are **durable**: they persist across restarts, so workflows can have very long timeouts (hours, days, weeks)
- Cancellation happens at the **beginning of the next step** - the current step completes first; the workflow's `Context` is also cancelled, so a running step can react to `ctx.Done()`
- Timeout cancellation propagates to all **child workflows**. A child started under its parent's context inherits the parent's deadline from the start, even when enqueued (v1.6.0+); pass a shorter `WithTimeout` context to bound it further
- Timed-out workflows are cancelled by a monitor that every launched process runs, including workflows no process is executing. It polls once per second, so a `Context` may be cancelled shortly after its deadline

To set an absolute deadline and choose the error reported as the cancellation cause, use `dbos.WithDeadlineCause(ctx, deadline, cause)`, the counterpart of `context.WithDeadlineCause`. For manual cancellation without a deadline, derive a cancellable context with `dbos.WithCancel(ctx)` (or `dbos.WithCancelCause(ctx)`) and call the returned cancel function. Use `dbos.WithoutCancel(ctx)` to detach a child workflow from its parent's cancellation.

Reference: [Workflow Timeouts](https://docs.dbos.dev/golang/tutorials/workflow-tutorial#workflow-timeouts)
