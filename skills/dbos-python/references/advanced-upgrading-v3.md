---
title: Upgrade DBOS Python 2.x Code to 3.x
impact: HIGH
impactDescription: 3.0 removed deprecated APIs and changed the storage format; old code fails and mixed deployments break
tags: upgrade, migration, 3.0, breaking-changes, transaction, datasource, queue, scheduled, fastapi
---

## Upgrade DBOS Python 2.x Code to 3.x

DBOS Python 3.0 removed features deprecated in 2.x. When you see any API below in existing code, replace it. Never generate the "Incorrect" forms for 3.x.

### Rolling Out the Upgrade Safely

3.0 changes the storage format of workflow inputs and outputs. 3.0 can process workflows created by 2.x, but **2.x cannot process workflows created by 3.0**.

- Don't run 2.x and 3.0 processes concurrently with the same application version. If you set `application_version` yourself, change it when you upgrade.
- If you use patching, shut down all 2.x processes before launching 3.0 processes.
- Upgrade applications that use `DBOSClient` together with your DBOS processes: a 2.x client cannot read inputs or results of workflows created by 3.0.

### `@DBOS.transaction` → Datasources

`@DBOS.transaction`, `DBOS.sql_session`, and the `application_database_url` / `database_url` config fields were removed.

**Incorrect (removed in 3.0):**

```python
config: DBOSConfig = {
    "name": "my-app",
    "system_database_url": os.environ["DBOS_SYSTEM_DATABASE_URL"],
    "application_database_url": os.environ["APP_DATABASE_URL"],
}
DBOS(config=config)

@DBOS.transaction()
def insert_greeting(name: str, note: str) -> None:
    sql = text("INSERT INTO greetings (name, note) VALUES (:name, :note)")
    DBOS.sql_session.execute(sql, {"name": name, "note": note})
```

**Correct:**

```python
import os
from sqlalchemy import text
from dbos import DBOS, DBOSConfig, SQLAlchemyDatasource

config: DBOSConfig = {
    "name": "my-app",
    "system_database_url": os.environ["DBOS_SYSTEM_DATABASE_URL"],
}
DBOS(config=config)
ds = SQLAlchemyDatasource.create(os.environ["APP_DATABASE_URL"])  # before DBOS.launch()

@ds.transaction()
def insert_greeting(name: str, note: str) -> None:
    sql = text("INSERT INTO greetings (name, note) VALUES (:name, :note)")
    ds.sql_session().execute(sql, {"name": name, "note": note})
```

If the old config set `database_url`/`application_database_url` but not `system_database_url`, DBOS state lived in a separate database: on Postgres, the app database name plus `_dbos_sys`; on SQLite, the same file. Set `system_database_url` to it. See [step-transactions](step-transactions.md).

### In-Memory `Queue(...)` → `DBOS.register_queue`

**Incorrect (removed in 3.0):**

```python
from dbos import DBOS, Queue

queue = Queue("example_queue", worker_concurrency=5)
DBOS.launch()
handle = queue.enqueue(process_task, task)
```

**Correct:**

```python
DBOS.launch()
DBOS.register_queue("example_queue", worker_concurrency=5)  # async: await DBOS.register_queue_async(...)
handle = DBOS.enqueue_workflow("example_queue", process_task, task)
```

- Register every queue the app previously declared; workflows on an unregistered queue stay `ENQUEUED`.
- `DBOS.listen_queues` accepts only queue names.
- Queue names starting with `_dbos_` are reserved.

### Legacy Partitioned Queues and `priority_enabled`

**Incorrect (removed in 3.0):**

```python
DBOS.register_queue("partitioned_queue", partition_queue=True, concurrency=1)
DBOS.register_queue("priority_queue", priority_enabled=True)
```

**Correct:**

```python
DBOS.register_queue("partitioned_queue", partition_concurrency=1)
DBOS.register_queue("priority_queue")  # priority is always enabled
```

With `partition_queue=True`, `concurrency`/`worker_concurrency`/`limiter` applied per partition; map them to `partition_concurrency`/`partition_worker_concurrency`/`partition_limiter`. The `Queue` methods for reading and setting `priority_enabled` and `partition_queue` were also removed. See [queue-partitioning](queue-partitioning.md).

### `@DBOS.scheduled` → Schedule API

The second argument of a scheduled workflow is now the schedule's `context`, not the actual start time.

**Incorrect (removed in 3.0):**

```python
from datetime import datetime

@DBOS.scheduled("*/5 * * * *")
@DBOS.workflow()
def my_periodic_task(scheduled_time: datetime, actual_time: datetime):
    ...
```

**Correct:**

```python
from datetime import datetime
from typing import Any

@DBOS.workflow()
def my_periodic_task(scheduled_time: datetime, context: Any):
    ...

DBOS.launch()
DBOS.apply_schedules([
    {"schedule_name": "my-periodic-task", "workflow_fn": my_periodic_task, "schedule": "*/5 * * * *"},
])
```

Schedules persist in the system database: one you stop applying keeps running until `DBOS.delete_schedule`. See [pattern-scheduled](pattern-scheduled.md).

### `DBOS(fastapi=...)` / `DBOS(flask=...)`

**Incorrect (removed in 3.0):**

```python
app = FastAPI()
DBOS(fastapi=app, config=config)
```

**Correct:** `DBOS(config=config)`, then call `DBOS.launch()` in `main` before `uvicorn.run`, or from a FastAPI lifespan (see [lifecycle-fastapi](lifecycle-fastapi.md)). For Flask, drop `flask=app` and call `DBOS.launch()` before starting the app. Use your framework's OpenTelemetry instrumentation for HTTP spans.

### Related Current Behavior

- `max_recovery_attempts` is not a `DBOSClient` `EnqueueOptions` field; set it on `@DBOS.workflow(max_recovery_attempts=...)`.
- The default workflow name is the function's `__qualname__`, which does not include its module. Registering workflows with the same name from different modules raises a `DBOSException`; pass `name=` to disambiguate.

Reference: [Upgrading to 3.0](https://docs.dbos.dev/python/upgrading)
