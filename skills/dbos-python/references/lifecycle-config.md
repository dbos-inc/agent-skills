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
| **application_database_url** | App DB for `@DBOS.transaction` | Same as system DB |
| **enable_patching** | Enable patching strategy for workflow upgrades | `False` |
| **application_version** | Version tag for versioning strategy | Auto-computed hash |
| **executor_id** | Unique process ID for distributed environments | Auto-set by Conductor |
| **sys_db_pool_size** | System DB connection pool size | `20` |
| **db_engine_kwargs** | Extra kwargs for SQLAlchemy `create_engine` | `None` |
| **dbos_system_schema** | Postgres schema for DBOS system tables | `"dbos"` |
| **system_database_engine** | Custom SQLAlchemy engine (skips engine creation) | `None` |
| **use_listen_notify** | Use Postgres LISTEN/NOTIFY vs polling | `True` (Postgres) |
| **notification_listener_polling_interval_sec** | Polling interval when LISTEN/NOTIFY is off | `1.0` |
| **enable_otlp** | Enable OpenTelemetry tracing and export | `False` |
| **otlp_traces_endpoints** | OTLP trace receiver URLs | `None` |
| **otlp_logs_endpoints** | OTLP log receiver URLs | `None` |
| **otlp_attributes** | Key-value pairs applied to all OTLP exports | `None` |
| **log_level** | DBOS logger severity | `"INFO"` |
| **otlp_log_level** | OTLP-specific log level (>= `log_level`) | `log_level` |
| **console_log_level** | Console-specific log level (>= `log_level`) | `log_level` |
| **run_admin_server** | Run HTTP admin server | `True` |
| **admin_port** | Admin server port | `3001` |
| **max_executor_threads** | Max threads for sync workflow/step execution | `None` |
| **scheduler_polling_interval_sec** | Scheduler polling interval for new schedules | `30.0` |
| **serializer** | Custom serializer for system database | Default (pickle) |

Reference: [DBOS Configuration](https://docs.dbos.dev/python/reference/configuration)
