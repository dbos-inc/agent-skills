---
title: Enqueue Workflows from External Applications
impact: HIGH
impactDescription: Enables decoupled architecture with separate API and worker services
tags: client, enqueue, workflow, external
---

## Enqueue Workflows from External Applications

Use `client.enqueue()` to submit workflows from outside the DBOS application. Must specify workflow and queue names explicitly.

**Incorrect (missing required options):**

```python
from dbos import DBOSClient

client = DBOSClient(system_database_url=db_url)

# Missing workflow_name and queue_name!
handle = client.enqueue({}, task_data)
```

**Correct (with required options):**

```python
from dbos import DBOSClient, EnqueueOptions

client = DBOSClient(system_database_url=db_url)

# Optionally register the queue from the client (persists to system database)
client.register_queue("task_queue", concurrency=10)

options: EnqueueOptions = {
    "workflow_name": "process_task",  # Required
    "queue_name": "task_queue",       # Required
}
handle = client.enqueue(options, task_data)
result = handle.get_result()
client.destroy()
```

The queue does not need to exist when `enqueue` is called. If no queue with the given name has been registered, the workflow is still durably recorded as `ENQUEUED` and starts running once the queue is registered and a worker becomes available.

Optional parameters:

```python
options: EnqueueOptions = {
    "workflow_name": "process_task",
    "queue_name": "task_queue",
    "workflow_id": "custom-id-123",
    "workflow_timeout": 300,
    "deduplication_id": "user-123",
    "priority": 1,
    "delay_seconds": 60,            # Delay before becoming eligible
    "queue_partition_key": "user-123",
    "app_version": "1.0.0",
    "max_recovery_attempts": 50,
    "authenticated_user": "alice",
    "authenticated_roles": ["admin"],
}
```

Limitation: Cannot enqueue workflows that are methods on Python classes.

Reference: [DBOSClient.enqueue](https://docs.dbos.dev/python/reference/client#enqueue)
