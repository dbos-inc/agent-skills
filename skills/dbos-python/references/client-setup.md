---
title: Initialize DBOSClient for External Access
impact: HIGH
impactDescription: Enables external applications to interact with DBOS
tags: client, setup, initialization, external, schedule, debounce, version
---

## Initialize DBOSClient for External Access

Use `DBOSClient` to interact with DBOS from external applications (API servers, CLI tools, etc.).

**Incorrect (no cleanup):**

```python
from dbos import DBOSClient

client = DBOSClient(system_database_url=db_url)
handle = client.enqueue(options, data)
# Connection leaked - no destroy()!
```

**Correct (with cleanup):**

```python
import os
from dbos import DBOSClient

client = DBOSClient(
    system_database_url=os.environ["DBOS_SYSTEM_DATABASE_URL"],
    application_name="my-app",  # the app that runs the workflows
)

try:
    handle = client.enqueue(options, data)
    result = handle.get_result()
finally:
    client.destroy()
```

Constructor parameters:
- `system_database_url`: Connection string to DBOS system database (required unless `system_database_engine` is given)
- `system_database_engine`: Custom SQLAlchemy engine (if provided, no engine is created)
- `dbos_system_schema`: Postgres schema for DBOS system tables (default: `"dbos"`)
- `serializer`: Must match the DBOS application's serializer (default: pickle)
- `application_name`: The application the client acts for. Enqueued workflows, registered queues, and created schedules are owned by it, and listing calls default to its rows. **Always set this if multiple applications share the system database.** See [advanced-shared-database](advanced-shared-database.md).
- `system_database_pool_size` (default 5), `system_database_polling_concurrency` (default half the pool)
- `use_listen_notify` (default `False`): use Postgres LISTEN to wake `get_event` / `read_stream` instead of polling
- `lazy` (default `False`): don't connect until first use; check explicitly with `client.check_connection()`. Cannot be combined with `use_listen_notify`
- `retry_connection_errors` (default `True`): block and retry on lost connections; `False` raises instead
- `observability_query_timeout_sec` (default 30): statement timeout for listing queries on Postgres

**Upgrading:** A DBOS 2.x client cannot read the inputs or results of workflows created by DBOS 3.0. Upgrade applications that use `DBOSClient` together with your DBOS processes (see [advanced-upgrading-v3](advanced-upgrading-v3.md)).

## API Reference

DBOSClient mirrors the DBOS API for workflow interaction:

| DBOSClient method | Same as DBOS method |
|-------------------|---------------------|
| `client.send()` | `DBOS.send()` - add `idempotency_key` for exactly-once |
| `client.send_bulk()` | `DBOS.send_bulk()` |
| `client.send_in_transaction()` / `client.send_bulk_in_transaction()` | Send atomically inside your own system-DB transaction (sync only) |
| `client.enqueue_in_transaction()` | Enqueue atomically inside your own system-DB transaction (sync only) |
| `client.get_event()` | `DBOS.get_event()` |
| `client.read_stream()` / `client.read_stream_offset()` | `DBOS.read_stream()` / `DBOS.read_stream_offset()` (client reads are never checkpointed) |
| `client.list_workflows()` | `DBOS.list_workflows()` |
| `client.list_queued_workflows()` | `DBOS.list_queued_workflows()` |
| `client.list_workflow_steps()` | `DBOS.list_workflow_steps()` |
| `client.cancel_workflow()` / `cancel_workflows()` | `DBOS.cancel_workflow()` / `cancel_workflows()` |
| `client.resume_workflow()` / `resume_workflows()` | `DBOS.resume_workflow()` / `resume_workflows()` |
| `client.retrieve_workflow()` | `DBOS.retrieve_workflow()` |
| `client.fork_workflow()` | `DBOS.fork_workflow()` |
| `client.rewind_workflow()` | `DBOS.rewind_workflow()` (does not delete datasource checkpoints; use `DBOS.rewind_workflow` for workflows using datasources) |
| `client.update_workflow_attributes()` | `DBOS.update_workflow_attributes()` |
| `client.delete_workflow()` / `delete_workflows()` | `DBOS.delete_workflow()` / `delete_workflows()` |
| `client.wait_first()` | `DBOS.wait_first()` |
| `client.set_workflow_delay()` | `DBOS.set_workflow_delay()` |
| `client.register_queue()` | `DBOS.register_queue()` |
| `client.retrieve_queue()` | `DBOS.retrieve_queue()` |
| `client.list_queues()` | `DBOS.list_queues()` |
| `client.delete_queue()` | `DBOS.delete_queue()` |
| `client.check_connection()` | Raise if the system database is unreachable |
| `client.rename_application(old_name, new_name)` | Transfer ownership of all rows after renaming an app (stop the app first) |

Most methods have `_async` variants. The `*_in_transaction` methods, `destroy`, and `pause_schedule` / `resume_schedule` / `backfill_schedule` / `trigger_schedule` do not.

## Schedule Management

Manage workflow schedules from outside the DBOS application. Uses workflow names as strings instead of function references:

```python
client.create_schedule(
    schedule_name="my-task",
    workflow_name="my_periodic_task",
    schedule="*/5 * * * *",
    context="my context",
)

schedules = client.list_schedules(status="ACTIVE")
schedule = client.get_schedule("my-task")
client.pause_schedule("my-task")
client.resume_schedule("my-task")
client.delete_schedule("my-task")
client.apply_schedules([...])  # Atomic batch create/update
client.backfill_schedule("my-task", start, end)
handle = client.trigger_schedule("my-task")
```

## Debouncing

```python
from dbos import DBOSClient, DebouncerClient, EnqueueOptions

workflow_options: EnqueueOptions = {
    "workflow_name": "process_input",
    "queue_name": "process_input_queue",
}
debouncer = DebouncerClient(client, workflow_options)

def on_user_input(user_id, user_input):
    debouncer.debounce(user_id, 60, user_input)  # Wait 60s idle
```

## Version Management

```python
versions = client.list_application_versions()
latest = client.get_latest_application_version()
client.set_latest_application_version("1.0.0")  # Rollback
```

Reference: [DBOSClient](https://docs.dbos.dev/python/reference/client)
