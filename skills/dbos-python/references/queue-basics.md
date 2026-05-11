---
title: Use Queues for Concurrent Workflows
impact: HIGH
impactDescription: Queues provide managed concurrency and flow control
tags: queue, concurrency, enqueue, workflow, register_queue
---

## Use Queues for Concurrent Workflows

Queues run many workflows concurrently with managed flow control. Use them when you need to control how many workflows run at once.

Register queues with `DBOS.register_queue` **after** `DBOS.launch()`. Queue configuration is persisted to the system database, so all DBOS processes and clients connected to the same system database see it.

**Incorrect (uncontrolled concurrency):**

```python
@DBOS.workflow()
def process_task(task):
    pass

# Starting many workflows without control
for task in tasks:
    DBOS.start_workflow(process_task, task)  # Could overwhelm resources
```

**Incorrect (deprecated in-memory `Queue` constructor):**

```python
from dbos import Queue

# Deprecated: in-memory only, not visible to other processes or clients
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

`on_conflict` controls how `register_queue` handles an existing queue in the system database:
- `"update_if_latest_version"` (default): overwrite only if this app is the latest registered application version
- `"always_update"`: always overwrite
- `"never_update"`: leave existing configuration unchanged (use this if you reconfigured the queue at runtime via `set_*` methods)

Reference: [DBOS Queues](https://docs.dbos.dev/python/tutorials/queue-tutorial)
