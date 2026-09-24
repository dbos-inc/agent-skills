---
title: Deduplicate Queued Workflows
impact: HIGH
impactDescription: Prevents duplicate work and resource waste
tags: queue, deduplication, duplicate, idempotent, singleton, duplication_policy
---

## Deduplicate Queued Workflows

Use deduplication IDs to ensure only one workflow with a given ID is active in a queue at a time.

**Incorrect (duplicate workflows possible):**

```python
DBOS.register_queue("user_tasks")

@app.post("/process/{user_id}")
def process_for_user(user_id: str):
    # Multiple requests = multiple workflows for same user!
    DBOS.enqueue_workflow("user_tasks", process_workflow, user_id)
```

**Correct (deduplicated by user):**

```python
from dbos import DBOS, SetEnqueueOptions
from dbos import error as dboserror

DBOS.register_queue("user_tasks")  # after DBOS.launch()

@app.post("/process/{user_id}")
def process_for_user(user_id: str):
    with SetEnqueueOptions(deduplication_id=user_id):
        try:
            handle = DBOS.enqueue_workflow("user_tasks", process_workflow, user_id)
            return {"workflow_id": handle.workflow_id}
        except dboserror.DBOSQueueDeduplicatedError:
            return {"status": "already processing"}
```

Deduplication behavior:
- If a workflow with the same deduplication ID is `DELAYED`, `ENQUEUED`, or `PENDING`, new enqueue raises `DBOSQueueDeduplicatedError`
- Once the workflow completes, a new workflow with the same ID can be enqueued
- Deduplication is per-queue (same ID can exist in different queues)
- Deduplication is not supported on partitioned queues

Use cases:
- One active task per user
- Preventing duplicate job submissions
- Rate limiting by entity

### Singleton Workflows

To attach to the existing workflow instead of raising, set `duplication_policy="return-existing"` on `SetEnqueueOptions`. The colliding caller's arguments are discarded and the returned handle resolves with the original workflow's result:

```python
from dbos import DBOS, SetEnqueueOptions

DBOS.register_queue("reports")  # after DBOS.launch()
# build_report is a workflow defined elsewhere

# Only one "nightly-report" workflow is active on the queue at a time;
# concurrent callers all get a handle to the same workflow.
with SetEnqueueOptions(deduplication_id="nightly-report", duplication_policy="return-existing"):
    handle = DBOS.enqueue_workflow("reports", build_report)
result = handle.get_result()
```

- `duplication_policy` is `"reject"` (default, raises `DBOSQueueDeduplicatedError`) or `"return-existing"`
- `"return-existing"` requires a queue and a `deduplication_id`
- Once the original workflow completes, the ID is released and the next caller starts a new workflow
- `DBOSClient` supports it as `"duplication_policy": "return-existing"` in `EnqueueOptions` (not with `enqueue_in_transaction`)

Reference: [Queue Deduplication](https://docs.dbos.dev/python/tutorials/queue-tutorial#deduplication)
