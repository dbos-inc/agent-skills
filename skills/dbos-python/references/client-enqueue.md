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
client.register_queue("task_queue", global_concurrency=10)

options: EnqueueOptions = {
    "workflow_name": "process_task",  # Required
    "queue_name": "task_queue",       # Required
}
handle = client.enqueue(options, task_data)
result = handle.get_result()
client.destroy()
```

The queue does not need to exist when `enqueue` is called. If no queue with the given name has been registered, the workflow is still durably recorded as `ENQUEUED` and starts running once the queue is registered and a worker becomes available.

`workflow_name` is the workflow's registered name: the `name` passed to `@DBOS.workflow`, or by default the function's `__qualname__` (no module prefix, e.g. `process_task` or `URLFetcher.fetch_workflow`).

Optional parameters:

```python
options: EnqueueOptions = {
    "workflow_name": "process_task",
    "queue_name": "task_queue",
    "workflow_id": "custom-id-123",
    "workflow_id_reuse_policy": "reject",   # or "return-existing" (default)
    "workflow_timeout": 300,
    "deduplication_id": "user-123",
    "duplication_policy": "return-existing", # singleton: attach to the existing workflow
    "priority": 1,
    "delay_seconds": 60,                    # Delay before becoming eligible
    "queue_partition_key": "user-123",      # only for partitioned queues
    "app_version": "1.0.0",                 # unset = dequeued by the latest version
    "authenticated_user": "alice",
    "authenticated_roles": ["admin"],
    "attributes": {"customer": "acme"},     # searchable via list_workflows(attributes=...)
    "application_name": "order-service",    # owning app, if the system DB is shared
}
```

- `workflow_id_reuse_policy="reject"` raises `DBOSWorkflowIDInUseError` if the ID exists.
- `duplication_policy="return-existing"` requires `deduplication_id`; otherwise a collision raises `DBOSQueueDeduplicatedError`.
- Also available: `serialization_type` (see [advanced-serialization](advanced-serialization.md)) and `otel_context` (propagate an OpenTelemetry trace context).
- `max_recovery_attempts` is not an enqueue option (removed in 3.0); set it on `@DBOS.workflow(max_recovery_attempts=...)`.

### Enqueueing Class Methods

To enqueue a `@classmethod` workflow, set `class_name`; for a method on a configured instance, set both `class_name` and `instance_name` (the instance's `config_name`). The class and instance must be registered in the application that dequeues the workflow. Static methods need neither.

```python
options: EnqueueOptions = {
    "queue_name": "example_queue",
    "workflow_name": "URLFetcher.fetch_workflow",
    "class_name": "URLFetcher",
    "instance_name": "https://example.com",
}
handle = client.enqueue(options)
```

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
import os
import sqlalchemy as sa

# Target the DBOS system database. For Postgres, use the psycopg (v3) driver
# DBOS installs; a plain postgresql:// URL makes SQLAlchemy look for psycopg2.
engine = sa.create_engine(
    sa.make_url(os.environ["DBOS_SYSTEM_DATABASE_URL"]).set(drivername="postgresql+psycopg")
)

with engine.begin() as conn:
    # Your own writes...
    conn.execute(text("INSERT INTO orders (id) VALUES (:id)"), {"id": order_id})
    # Enqueue in the same transaction
    handle = client.enqueue_in_transaction(conn, options, task_data)
    # The workflow is enqueued only when this transaction commits

result = handle.get_result()  # Safe to call only after commit
```

- Like `enqueue` (except `duplication_policy="return-existing"` is not supported), but performs the enqueue write inside a caller-owned SQLAlchemy transaction, so the enqueue commits or rolls back atomically with your own DB writes.
- Pass a SQLAlchemy `Connection` or ORM `Session`. It must target the DBOS **system** database (the enqueue can't atomically span a separate app database).
- You own the transaction: the method does not begin/commit/roll back and does not retry on DB errors. You must commit yourself.
- The returned handle is created immediately, but the workflow is not enqueued until you commit—do not call `get_result()` until after commit.
- `client.send_in_transaction(conn, destination_id, message, topic, idempotency_key)` and `client.send_bulk_in_transaction(conn, messages)` send messages atomically with your writes the same way.
- No async variant; from async code, bridge via `AsyncConnection.run_sync(lambda sync_conn: client.enqueue_in_transaction(sync_conn, options, *args))`.

Reference: [DBOSClient.enqueue](https://docs.dbos.dev/python/reference/client#enqueue)
