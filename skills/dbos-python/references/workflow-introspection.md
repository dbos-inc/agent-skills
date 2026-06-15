---
title: List and Inspect Workflows
impact: MEDIUM
impactDescription: Enables monitoring and management of workflow state
tags: workflow, list, introspection, status, monitoring
---

## List and Inspect Workflows

Use `DBOS.list_workflows()` to query workflows by status, name, queue, or other criteria.

**Incorrect (loading unnecessary data):**

```python
# Loading inputs/outputs when not needed is slow
workflows = DBOS.list_workflows(status="PENDING")
for w in workflows:
    print(w.workflow_id)  # Only using ID
```

**Correct (optimize with load flags):**

```python
# Disable loading inputs/outputs for better performance
workflows = DBOS.list_workflows(
    status="PENDING",
    load_input=False,
    load_output=False
)
for w in workflows:
    print(f"{w.workflow_id}: {w.status}")
```

Common queries:

```python
# Find failed workflows
failed = DBOS.list_workflows(status="ERROR", limit=100)

# Find workflows by name, filtering on multiple statuses
processing = DBOS.list_workflows(
    name="process_task",
    status=["PENDING", "ENQUEUED"],
)

# Find workflows on a specific queue
queued = DBOS.list_workflows(queue_name="high_priority")

# Only queued workflows (shortcut)
queued = DBOS.list_queued_workflows(queue_name="task_queue")

# Find old-version workflows for blue-green deploys
old = DBOS.list_workflows(
    app_version="1.0.0",
    status=["PENDING", "ENQUEUED"],
)

# Find children of a parent workflow
children = DBOS.list_workflows(parent_workflow_id=parent_id)

# Find every workflow forked from one ID
forks = DBOS.list_workflows(forked_from=original_id)

# Sort newest-first, paginate
page = DBOS.list_workflows(limit=50, offset=100, sort_desc=True)
```

### Filter Parameters

`list_workflows` accepts:

- **workflow_ids**: List of specific IDs to fetch
- **status**: Single status or list (see status values below)
- **start_time** / **end_time**: RFC 3339 timestamps
- **completed_after**: Retrieve workflows that completed after this RFC 3339 timestamp
- **completed_before**: Retrieve workflows that completed before this RFC 3339 timestamp
- **dequeued_after**: Retrieve workflows that were dequeued after this RFC 3339 timestamp
- **dequeued_before**: Retrieve workflows that were dequeued before this RFC 3339 timestamp
- **name**: Fully-qualified workflow function name (or list)
- **app_version**: Application version(s)
- **forked_from**: Source workflow ID(s) for forks
- **parent_workflow_id**: Parent workflow ID(s)
- **user**: Authenticated user(s)
- **queue_name**: Queue name(s)
- **workflow_id_prefix**: Match workflows whose IDs start with this
- **executor_id**: Executor ID(s) the workflow ran on
- **limit** / **offset**: Pagination
- **sort_desc**: Sort by start time descending (default ascending)
- **load_input** / **load_output**: Set to `False` to skip deserializing for performance
- **queues_only**: If `True`, only `ENQUEUED`/`PENDING` workflows on a queue (same as `list_queued_workflows`)
- **has_parent**: `True` for workflows with a parent, `False` for top-level only
- **was_forked_from**: `True` for workflows that have been forked from, `False` for those that haven't

### Status Values

`ENQUEUED`, `DELAYED`, `PENDING`, `SUCCESS`, `ERROR`, `CANCELLED`, `MAX_RECOVERY_ATTEMPTS_EXCEEDED`

- `ENQUEUED`: durably recorded on a queue, awaiting dequeue
- `DELAYED`: enqueued with a `delay_seconds`; transitions to `ENQUEUED` when the delay expires
- `PENDING`: actively executing (or about to)
- `SUCCESS` / `ERROR`: terminal
- `CANCELLED`: cancelled via `cancel_workflow` (or timed out)
- `MAX_RECOVERY_ATTEMPTS_EXCEEDED`: exceeded retry attempts on recovery

`cancel_workflow(workflow_id, *, cancel_children=False)` and `cancel_workflows(workflow_ids, *, cancel_children=False)` cancel only the named workflows by default. Set `cancel_children=True` to also recursively cancel all child workflows.

### WorkflowStatus Fields

```python
class WorkflowStatus:
    workflow_id: str
    status: str
    name: str
    class_name: Optional[str]
    config_name: Optional[str]
    authenticated_user: Optional[str]
    assumed_role: Optional[str]
    authenticated_roles: Optional[list[str]]
    input: Optional[WorkflowInputs]
    output: Optional[Any]
    error: Optional[Exception]
    created_at: Optional[int]            # Unix epoch ms
    updated_at: Optional[int]
    queue_name: Optional[str]
    executor_id: Optional[str]
    app_version: Optional[str]
    workflow_timeout_ms: Optional[int]
    workflow_deadline_epoch_ms: Optional[int]
    deduplication_id: Optional[str]
    priority: Optional[int]
    queue_partition_key: Optional[str]
    forked_from: Optional[str]
    was_forked_from: bool
    parent_workflow_id: Optional[str]
    dequeued_at: Optional[int]
    completed_at: Optional[int]          # Unix epoch ms at which the workflow
                                         # completed (SUCCESS, ERROR, or CANCELLED),
                                         # if it has completed
```

### Listing Steps

```python
steps = DBOS.list_workflow_steps(workflow_id, limit=100, offset=0)
for step in steps:
    print(step["function_id"], step["function_name"])
```

Each `StepInfo` exposes: `function_id`, `function_name`, `output`, `error`, `child_workflow_id`, `started_at_epoch_ms`, `completed_at_epoch_ms`.

Reference: [Workflow Management](https://docs.dbos.dev/python/tutorials/workflow-management)
