---
title: Partition Queues for Per-Entity Limits
impact: HIGH
impactDescription: Enables per-user or per-entity flow control
tags: queue, partition, per-user, flow-control
---

## Partition Queues for Per-Entity Limits

Partitioned queues apply flow control limits per partition, not globally. Useful for per-user or per-entity concurrency limits.

**Incorrect (global limit affects all users):**

```python
DBOS.register_queue("user_tasks", concurrency=1)  # Only 1 task total

def handle_user_task(user_id, task):
    # One user blocks all other users!
    DBOS.enqueue_workflow("user_tasks", process_task, task)
```

**Correct (per-user limits with partitioning):**

```python
from dbos import DBOS, SetEnqueueOptions

# Partition queue with concurrency=1 per partition
DBOS.register_queue("user_tasks", partition_queue=True, concurrency=1)

@DBOS.workflow()
def process_task(task):
    pass

def handle_user_task(user_id: str, task):
    # Each user gets their own "subqueue" with concurrency=1
    with SetEnqueueOptions(queue_partition_key=user_id):
        DBOS.enqueue_workflow("user_tasks", process_task, task)
```

For both per-partition AND global limits, use two-level queueing:

```python
# Worker concurrency limit of 5, meaning that no more than 5 tasks can run per worker
DBOS.register_queue("global_queue", worker_concurrency=5)
# Per-user limit of 1 concurrent task
DBOS.register_queue("user_queue", partition_queue=True, concurrency=1)

def handle_task(user_id: str, task):
    with SetEnqueueOptions(queue_partition_key=user_id):
        DBOS.enqueue_workflow("user_queue", concurrency_manager, task)

@DBOS.workflow()
def concurrency_manager(task):
    # Enforces worker concurrency limit
    return DBOS.enqueue_workflow("global_queue", process_task, task).get_result()

@DBOS.workflow()
def process_task(task):
    pass
```

Reference: [Partitioning Queues](https://docs.dbos.dev/python/tutorials/queue-tutorial#partitioning-queues)
