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
    system_database_url=os.environ["DBOS_SYSTEM_DATABASE_URL"]
)

try:
    handle = client.enqueue(options, data)
    result = handle.get_result()
finally:
    client.destroy()
```

Constructor parameters:
- `system_database_url`: Connection string to DBOS system database
- `system_database_engine`: Custom SQLAlchemy engine (if provided, no engine is created)
- `dbos_system_schema`: Postgres schema for DBOS system tables (default: `"dbos"`)
- `serializer`: Must match the DBOS application's serializer (default: pickle)

## API Reference

DBOSClient mirrors the DBOS API for workflow interaction:

| DBOSClient method | Same as DBOS method |
|-------------------|---------------------|
| `client.send()` | `DBOS.send()` - add `idempotency_key` for exactly-once |
| `client.get_event()` | `DBOS.get_event()` |
| `client.read_stream()` | `DBOS.read_stream()` |
| `client.list_workflows()` | `DBOS.list_workflows()` |
| `client.list_queued_workflows()` | `DBOS.list_queued_workflows()` |
| `client.list_workflow_steps()` | `DBOS.list_workflow_steps()` |
| `client.cancel_workflow()` | `DBOS.cancel_workflow()` |
| `client.resume_workflow()` | `DBOS.resume_workflow()` |
| `client.retrieve_workflow()` | `DBOS.retrieve_workflow()` |
| `client.fork_workflow()` | `DBOS.fork_workflow()` |
| `client.wait_first()` | `DBOS.wait_first()` |

All methods have `_async` variants.

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
