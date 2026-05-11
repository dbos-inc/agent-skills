---
title: Control Queue Concurrency
impact: HIGH
impactDescription: Prevents resource exhaustion with concurrent limits
tags: queue, concurrency, worker_concurrency, limits
---

## Control Queue Concurrency

Queues support worker-level and global concurrency limits to prevent resource exhaustion.

**Incorrect (no concurrency control):**

```python
DBOS.register_queue("heavy_tasks")  # No limits - could exhaust memory

@DBOS.workflow()
def memory_intensive_task(data):
    # Uses lots of memory
    pass
```

**Correct (worker concurrency):**

```python
# Each process runs at most 5 tasks from this queue
DBOS.register_queue("heavy_tasks", worker_concurrency=5)

@DBOS.workflow()
def memory_intensive_task(data):
    pass
```

**Correct (global concurrency):**

```python
# At most 10 tasks run across ALL processes
DBOS.register_queue("limited_tasks", concurrency=10)
```

**In-order processing (sequential):**

```python
# Only one task at a time - guarantees order
DBOS.register_queue("sequential_queue", concurrency=1)

@DBOS.step()
def process_event(event):
    pass

def handle_event(event):
    DBOS.enqueue_workflow("sequential_queue", process_event, event)
```

Worker concurrency is recommended for most use cases. Global concurrency should be used carefully as pending workflows count toward the limit (including workflows from previous application versions).

When using worker concurrency, each process must have a unique `executor_id` set in configuration (this is automatic with DBOS Conductor or Cloud).

### Reconfiguring at Runtime

Because queue configuration lives in the system database, you can change a queue's concurrency at runtime without redeploying. Workers pick up the new configuration on their next polling iteration.

```python
queue = DBOS.retrieve_queue("heavy_tasks")
queue.set_concurrency(20)
queue.set_worker_concurrency(2)
```

In `async` code, use the `_async` variants (`set_concurrency_async`, `set_worker_concurrency_async`) to avoid blocking the event loop.

If your application also calls `DBOS.register_queue` on startup, the next process to launch can overwrite your runtime changes. Either update the `register_queue` call to match, or pass `on_conflict="never_update"` to preserve runtime values.

Reference: [Managing Concurrency](https://docs.dbos.dev/python/tutorials/queue-tutorial#managing-concurrency)
