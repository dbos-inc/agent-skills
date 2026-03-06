---
title: Start Workflows in Background
impact: CRITICAL
impactDescription: Background workflows survive crashes and restarts
tags: workflow, background, start_workflow, handle, wait_first
---

## Start Workflows in Background

Use `DBOS.start_workflow` to run workflows in the background. This returns a handle to monitor or retrieve results.

**Incorrect (using threads):**

```python
import threading

@DBOS.workflow()
def long_task(data):
    # Long running work
    pass

# Don't use threads for DBOS workflows!
thread = threading.Thread(target=long_task, args=(data,))
thread.start()
```

**Correct (using start_workflow):**

```python
from dbos import DBOS, WorkflowHandle

@DBOS.workflow()
def long_task(data):
    # Long running work
    return "done"

# Start workflow in background
handle: WorkflowHandle = DBOS.start_workflow(long_task, data)

# Later, get the result
result = handle.get_result()

# Or check status
status = handle.get_status()
```

You can retrieve a workflow handle later using its ID:

```python
# Get workflow ID
workflow_id = handle.get_workflow_id()

# Later, retrieve the handle
handle = DBOS.retrieve_workflow(workflow_id)
result = handle.get_result()
```

### Waiting for the First of Multiple Workflows

Use `DBOS.wait_first` to race multiple concurrent workflows and process results as they complete:

```python
handles = [
    DBOS.start_workflow(process_item, item)
    for item in items
]

# Wait for whichever finishes first
first_done = DBOS.wait_first(handles)
result = first_done.get_result()
```

`wait_first` parameters:
- `handles`: A non-empty list of `WorkflowHandle` to wait on (raises `ValueError` if empty)
- `polling_interval_sec`: Polling interval in seconds (default `1.0`)

For async workflows, use `DBOS.start_workflow_async` and `DBOS.wait_first_async`.

Reference: [Starting Workflows](https://docs.dbos.dev/python/tutorials/workflow-tutorial#starting-workflows-in-the-background)
