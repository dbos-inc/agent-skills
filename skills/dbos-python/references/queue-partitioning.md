---
title: Partition Queues for Per-Entity Limits
impact: HIGH
impactDescription: Enables per-user or per-entity flow control
tags: queue, partition, per-user, flow-control, partition_concurrency
---

## Partition Queues for Per-Entity Limits

A queue is partitioned if you register it with any per-partition limit (`partition_concurrency`, `partition_worker_concurrency`, or `partition_limiter`). Each partition key acts as a dynamically created "subqueue". Useful for per-user or per-tenant limits.

**Incorrect (global limit affects all users):**

```python
DBOS.register_queue("user_tasks", global_concurrency=1)  # Only 1 task total

def handle_user_task(user_id, task):
    # One user blocks all other users!
    DBOS.enqueue_workflow("user_tasks", process_task, task)
```

**Incorrect (legacy `partition_queue=True`, removed in 3.0):**

```python
DBOS.register_queue("user_tasks", partition_queue=True, concurrency=1)
```

**Correct (per-user limits with partitioning):**

```python
from dbos import DBOS, SetEnqueueOptions

@DBOS.workflow()
def process_task(task):
    pass

# At most one task at a time per user
DBOS.register_queue("user_tasks", partition_concurrency=1)

def handle_user_task(user_id: str, task):
    # Every enqueue on a partitioned queue MUST supply a partition key
    with SetEnqueueOptions(queue_partition_key=user_id):
        DBOS.enqueue_workflow("user_tasks", process_task, task)
```

### Combining Queue-Wide and Per-Partition Limits

A partitioned queue enforces its per-partition limits **and** its queue-wide limits (`global_concurrency`, `worker_concurrency`, `limiter`) at the same time:

```python
# "Fair queue": at most 1 task per user, at most 10 tasks per process
DBOS.register_queue("fair_queue", partition_concurrency=1, worker_concurrency=10)

# Mix and match freely
DBOS.register_queue(
    "tenant_queue",
    global_concurrency=100,                          # across all processes
    worker_concurrency=10,                           # per process
    limiter={"limit": 1000, "period": 60},           # starts per minute, whole queue
    partition_concurrency=25,                        # per tenant, across all processes
    partition_worker_concurrency=2,                  # per tenant, per process
    partition_limiter={"limit": 50, "period": 60},   # starts per minute, per tenant
)
```

Rules:
- Each per-partition concurrency limit must be <= its queue-wide counterpart, and `partition_worker_concurrency` <= `partition_concurrency`
- A workflow enqueued on a partitioned queue **without** a partition key stays `ENQUEUED` and is never dequeued
- Deduplication (`deduplication_id`) is not supported on partitioned queues
- Setting a `partition_*` limit at runtime (`queue.set_partition_concurrency(...)`) partitions the queue; already-enqueued workflows without a key will not be dequeued until it is unpartitioned

Reference: [Partitioning Queues](https://docs.dbos.dev/python/tutorials/queue-tutorial#partitioning-queues)
