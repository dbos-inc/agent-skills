---
title: Use Queues for Concurrent Workflows
impact: HIGH
impactDescription: Queues provide managed concurrency and flow control
tags: queue, concurrency, enqueue, workflow, register_queue, priority
---

## Use Queues for Concurrent Workflows

Queues run many workflows concurrently with managed flow control. Use them when you need to control how many workflows run at once.

Register queues with `DBOS.register_queue` **after** `DBOS.launch()` (it raises if DBOS is not launched; in async code use `await DBOS.register_queue_async(...)`). Queue configuration is persisted to the system database, so all DBOS processes and clients connected to the same system database see it.

**Incorrect (uncontrolled concurrency):**

```python
@DBOS.workflow()
def process_task(task):
    pass

# Starting many workflows without control
for task in tasks:
    DBOS.start_workflow(process_task, task)  # Could overwhelm resources
```

**Incorrect (in-memory `Queue` constructor, removed in 3.0):**

```python
from dbos import Queue

# Removed in 3.0: raises DBOSException
queue = Queue("task_queue")
```

**Correct (database-backed queue):**

```python
from dbos import DBOS

@DBOS.workflow()
def process_task(task):
    pass

@DBOS.workflow()
def process_all_tasks(tasks):
    handles = []
    for task in tasks:
        # Enqueue by queue name
        handle = DBOS.enqueue_workflow("task_queue", process_task, task)
        handles.append(handle)
    # Wait for all tasks
    return [h.get_result() for h in handles]

if __name__ == "__main__":
    DBOS(config=config)
    DBOS.launch()
    # Register queues AFTER launch
    DBOS.register_queue("task_queue")
```

`DBOS.register_queue` returns a `Queue` object you can also call directly:

```python
queue = DBOS.register_queue("task_queue")
handle = queue.enqueue(process_task, task)
```

Queues process workflows in FIFO order. You can enqueue both workflows and steps.

- Enqueueing on a queue that has not been registered is allowed: the workflow stays `ENQUEUED` until a queue with that name is registered.
- Queue names must be unique in the system database (including across applications sharing it). Names starting with `_dbos_` are reserved.
- A queue is owned by the application that registers it; only that application dequeues from it.

`on_conflict` controls how `register_queue` handles an existing queue in the system database:
- `"update_if_latest_version"` (default): overwrite only if this app is the latest registered application version
- `"always_update"`: always overwrite
- `"never_update"`: leave existing configuration unchanged (use this if you reconfigured the queue at runtime via `set_*` methods)

### Priority

Priority is always enabled on every queue; set it per enqueue with `SetEnqueueOptions`. Lower numbers run first.

```python
from dbos import DBOS, SetEnqueueOptions

DBOS.register_queue("tasks")

def enqueue_task(task, is_urgent: bool):
    # Priority 1 = highest, runs before priority 10
    with SetEnqueueOptions(priority=1 if is_urgent else 10):
        DBOS.enqueue_workflow("tasks", process_task, task)
```

- Range: 1 to 2,147,483,647 (lower = higher priority)
- Workflows without a priority have the highest priority (run first)
- Same priority = FIFO order
- The `priority_enabled` parameter was removed in 3.0; do not pass it

Reference: [DBOS Queues](https://docs.dbos.dev/python/tutorials/queue-tutorial)
