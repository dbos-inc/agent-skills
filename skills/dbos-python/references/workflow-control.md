---
title: Cancel, Resume, and Fork Workflows
impact: MEDIUM
impactDescription: Control running workflows and recover from failures
tags: workflow, cancel, resume, fork, control
---

## Cancel, Resume, and Fork Workflows

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

Reference: [Workflow Management](https://docs.dbos.dev/python/tutorials/workflow-management)
