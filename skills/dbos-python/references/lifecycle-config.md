---
title: Configure and Launch DBOS Properly
impact: CRITICAL
impactDescription: Application won't function without proper setup
tags: configuration, launch, setup, initialization
---

## Configure and Launch DBOS Properly

Every DBOS application must configure and launch DBOS inside the main function.

**Incorrect (configuration at module level):**

```python
from dbos import DBOS, DBOSConfig

# Don't configure at module level!
config: DBOSConfig = {
    "name": "my-app",
}
DBOS(config=config)

@DBOS.workflow()
def my_workflow():
    pass

if __name__ == "__main__":
    DBOS.launch()
    my_workflow()
```

**Correct (configuration in main):**

```python
import os
from dbos import DBOS, DBOSConfig

@DBOS.workflow()
def my_workflow():
    pass

if __name__ == "__main__":
    config: DBOSConfig = {
        "name": "my-app",
        "system_database_url": os.environ.get("DBOS_SYSTEM_DATABASE_URL"),
    }
    DBOS(config=config)
    DBOS.launch()
    my_workflow()
```

For scheduled-only applications (no HTTP server), block the main thread:

```python
if __name__ == "__main__":
    DBOS(config=config)
    DBOS.launch()
    DBOS.apply_schedules([{
        "schedule_name": "my-task",
        "workflow_fn": scheduled_task,
        "schedule": "* * * * *",
    }])
    threading.Event().wait()  # Block forever
```

## DBOSConfig Reference

All fields except `name` are optional:

| Field | Description | Default |
|-------|-------------|---------|
| **name** | Application name | (required) |
| **system_database_url** | System DB connection string (Postgres or SQLite) | `sqlite:///[name].sqlite` |
| **enable_patching** | Enable patching strategy for workflow upgrades | `False` |
| **application_version** | Version tag for versioning strategy | Auto-computed hash |
| **executor_id** | Unique process ID for distributed environments | Auto-set by Conductor |
| **sys_db_pool_size** | System DB connection pool size | `20` |
| **db_engine_kwargs** | Extra kwargs for SQLAlchemy `create_engine` | `None` |
| **dbos_system_schema** | Postgres schema for DBOS system tables | `"dbos"` |
| **system_database_engine** | Custom SQLAlchemy engine (skips engine creation) | `None` |
| **use_listen_notify** | Use Postgres LISTEN/NOTIFY vs polling | `True` (Postgres) |
| **notification_listener_polling_interval_sec** | Polling interval when LISTEN/NOTIFY is off | `1.0` |
| **conductor_key** | API key for DBOS Conductor (from console.dbos.dev) | `None` |
| **conductor_url** | Conductor service URL (only for self-hosted) | `None` |
| **conductor_executor_metadata** | JSON dict of metadata sent to Conductor (region, instance type, etc.) | `None` |
| **enable_otlp** | Enable OpenTelemetry tracing and export | `False` |
| **otlp_traces_endpoints** | OTLP trace receiver URLs | `None` |
| **otlp_logs_endpoints** | OTLP log receiver URLs | `None` |
| **otlp_attributes** | Key-value pairs applied to all OTLP exports | `None` |
| **otel_attribute_format** | `"legacy"` (camelCase) or `"semconv"` (`dbos.*` namespace) | `"legacy"` |
| **log_level** | DBOS logger severity | `"INFO"` |
| **otlp_log_level** | OTLP-specific log level (>= `log_level`) | `log_level` |
| **console_log_level** | Console-specific log level (>= `log_level`) | `log_level` |
| **run_admin_server** | Run HTTP admin server | `True` |
| **admin_port** | Admin server port | `3001` |
| **max_executor_threads** | Max threads for sync workflow/step execution | `None` |
| **scheduler_polling_interval_sec** | Scheduler polling interval for new schedules | `30.0` |
| **serializer** | Custom serializer for system database | Default (pickle) |

## Lifecycle Methods

### Listening to Specific Queues

Use `DBOS.listen_queues` **before** `DBOS.launch()` to restrict a process to dequeuing from specific queues only (useful for heterogeneous worker pools). Pass queue names as strings or `Queue` objects:

```python
if __name__ == "__main__":
    DBOS(config=config)
    DBOS.listen_queues(["gpu_queue"])   # GPU worker
    DBOS.launch()
    DBOS.register_queue("cpu_queue")
    DBOS.register_queue("gpu_queue")
```

A process can still **enqueue** to any queue; `listen_queues` only controls dequeueing. See [queue-listening](queue-listening.md) for details.

### Tearing Down DBOS

`DBOS.destroy` shuts down the singleton (close connections, cancel polling, etc.) so it can be re-initialized — primarily used in tests.

```python
DBOS.destroy(
    workflow_completion_timeout_sec=30,   # Wait up to 30s for active workflows
    destroy_registry=False,               # Keep decorator registrations across destroy
)
```

Set `destroy_registry=True` only if you also want to un-register all decorated functions. Leave it `False` for normal teardown.

`DBOS.reset_system_database()` wipes the system DB's internal state — **destructive, test-only**.

## Connection Poolers (PgBouncer, PlanetScale, Supabase, Neon)

When connecting through a connection pooler in **transaction mode**, set `use_listen_notify` to `False`:

```python
config: DBOSConfig = {
    "name": "my-app",
    "system_database_url": os.environ.get("DBOS_SYSTEM_DATABASE_URL"),
    "use_listen_notify": False,
}
```

**Why:** Postgres LISTEN is connection-scoped state — the registration lives in the backend process's memory and is tied to the TCP connection. Transaction-mode poolers (PgBouncer, PlanetScale, Supabase Supavisor, Neon) return server connections to the pool after each transaction, orphaning the LISTEN registration. Subsequent NOTIFY messages are delivered to the server connection, but the pooler has no client mapped to forward them to — so notifications are **silently discarded**.

**Symptom:** `DBOS.recv()` and `DBOS.get_event()` block indefinitely with no errors.

**Fallback behavior:** With `use_listen_notify: False`, DBOS polls the `dbos.notifications` table every 1 second (configurable via `notification_listener_polling_interval_sec`). This adds up to 1 second of latency to message/event delivery but has negligible impact on database load since the query hits an indexed lookup.

**Session-mode poolers** (PgBouncer in session mode) maintain a 1:1 client-to-server mapping for the connection lifetime, so LISTEN/NOTIFY works normally. Only transaction-mode and statement-mode poolers require this setting.

Reference: [DBOS Configuration](https://docs.dbos.dev/python/reference/configuration)
