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

### Enqueue Inside Your Own Transaction

Use `client.enqueue_in_transaction()` to make the enqueue commit or roll back atomically with your own database writes:

```python
client.enqueue_in_transaction(
    conn_or_session: Union[sqlalchemy.Connection, sqlalchemy.orm.Session],
    options: EnqueueOptions,
    *args, **kwargs
) -> WorkflowHandle[R]
```

```python
with engine.begin() as conn:  # engine for the DBOS system database
    # Your own writes...
    conn.execute(text("INSERT INTO orders (id) VALUES (:id)"), {"id": order_id})
    # Enqueue in the same transaction
    handle = client.enqueue_in_transaction(conn, options, task_data)
    # The workflow is enqueued only when this transaction commits

result = handle.get_result()  # Safe to call only after commit
```

- Like `enqueue`, but performs the enqueue write inside a caller-owned SQLAlchemy transaction, so the enqueue commits or rolls back atomically with your own DB writes.
- Pass a SQLAlchemy `Connection` or ORM `Session`. It must target the DBOS **system** database (the enqueue can't atomically span a separate app database).
- You own the transaction: the method does not begin/commit/roll back and does not retry on DB errors. You must commit yourself.
- The returned handle is created immediately, but the workflow is not enqueued until you commit—do not call `get_result()` until after commit.
- No async variant; from async code, bridge via `AsyncConnection.run_sync(lambda sync_conn: client.enqueue_in_transaction(sync_conn, options, *args))`.

Reference: [DBOSClient.enqueue](https://docs.dbos.dev/python/reference/client#enqueue)
