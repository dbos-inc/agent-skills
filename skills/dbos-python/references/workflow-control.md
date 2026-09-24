---
title: Cancel, Resume, Fork, and Rewind Workflows
impact: MEDIUM
impactDescription: Control running workflows and recover from failures
tags: workflow, cancel, resume, fork, rewind, control
---

## Cancel, Resume, Fork, and Rewind Workflows

Use these methods to control workflow execution: stop runaway workflows, retry failed ones, or restart from a specific step.

**Incorrect (expecting immediate cancellation):**

```python
DBOS.cancel_workflow(workflow_id)
# Wrong: assuming the workflow stopped immediately
cleanup_resources()  # May race with workflow still running its current step
```

**Correct (wait for cancellation to complete):**

```python
DBOS.cancel_workflow(workflow_id)
# Cancellation happens at the START of the next step
# Wait for workflow to actually stop
handle = DBOS.retrieve_workflow(workflow_id)
status = handle.get_status()
while status.status == "PENDING":
    time.sleep(0.5)
    status = handle.get_status()
# Now safe to clean up
cleanup_resources()
```

### Cancel

Stop a workflow and remove it from its queue:

```python
DBOS.cancel_workflow(workflow_id)  # Cancels this workflow only, NOT its children

# Opt in to recursively cancel all child workflows
DBOS.cancel_workflow(workflow_id, cancel_children=True)
```

By default `cancel_workflow` cancels only the named workflow. Pass `cancel_children=True` to also recursively cancel every child workflow it started. The same `cancel_children` kwarg is available on `cancel_workflows(workflow_ids, *, cancel_children=False)`.

Cancellation interrupts the workflow at the **beginning of its next step**; a step that is already executing runs to completion first. To cancel an executing async step immediately, mark it [`preemptible`](step-basics.md) (`@DBOS.step(preemptible=True)`).

### Resume

Restart a stopped workflow from its last completed step:

```python
# Resume a cancelled or failed workflow
handle = DBOS.resume_workflow(workflow_id)
result = handle.get_result()

# Can also bypass queue for an enqueued workflow
handle = DBOS.resume_workflow(enqueued_workflow_id)
```

### Fork

Start a new workflow from a specific step of an existing one:

```python
# Get steps to find the right starting point
steps = DBOS.list_workflow_steps(workflow_id)
for step in steps:
    print(f"Step {step['function_id']}: {step['function_name']}")

# Fork from step 3 (skips steps 1-2, uses their saved results)
new_handle = DBOS.fork_workflow(workflow_id, start_step=3)

# Fork to run on a new application version (useful for patching bugs)
new_handle = DBOS.fork_workflow(
    workflow_id,
    start_step=3,
    application_version="2.0.0"
)
```

### Rewind

`DBOS.rewind_workflow` re-executes a workflow from a step **in place, keeping its workflow ID**. Unlike fork, other code that refers to the ID (senders, event/stream readers, idempotency keys) keeps working, and child workflows keep their IDs.

**Incorrect (forking when callers depend on the original ID):**

```python
# The order's workflow ID is "order-123"; forking gives it a new ID,
# so webhooks sending to "order-123" never reach the new execution.
DBOS.fork_workflow("order-123", start_step=3)
```

**Correct (rewind in place):**

```python
# Only terminal workflows (SUCCESS, ERROR, CANCELLED, MAX_RECOVERY_ATTEMPTS_EXCEEDED)
# can be rewound; cancel a running workflow first.
handle = DBOS.rewind_workflow("order-123", start_step=3)
result = handle.get_result()

# Omit start_step to re-run from the beginning; optionally move to a fixed version
handle = DBOS.rewind_workflow("order-123", application_version="2.0.0")
```

Rewinding discards steps with ID >= `start_step`, clears the output/error, restores events to their values before `start_step`, deletes messages consumed at or after `start_step` (and unconsumed ones), and deletes datasource transaction checkpoints at or after `start_step`. Streams are not truncated (new values append; streams closed at or after `start_step` are reopened), and child workflows are not modified. `queue_name` / `queue_partition_key` enqueue the rewound workflow on a specific queue (default: an internal queue that runs it immediately). Use `rewind_workflow_async` in async code.

`client.rewind_workflow` / `client.rewind_workflow_async` take the same arguments but do **not** delete datasource transaction checkpoints, so rewound datasource transactions are not re-executed. For workflows that use datasources, rewind with `DBOS.rewind_workflow` from a DBOS process.

Reference: [Workflow Management](https://docs.dbos.dev/python/tutorials/workflow-management)
