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
    "application_version": "0.1.0",
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
        "application_version": "0.1.0",
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
| **name** | Application name (see rules below) | (required) |
| **system_database_url** | System DB connection string (Postgres or SQLite). Postgres always uses the psycopg (v3) driver | `sqlite:///[name].sqlite` |
| **enable_patching** | Enable patching strategy for workflow upgrades | `False` |
| **application_version** | Version tag for versioning strategy. Set to `"0.1.0"` in new applications | Auto-computed hash |
| **executor_id** | Unique process ID for distributed environments | Auto-set by Conductor |
| **sys_db_pool_size** | System DB connection pool size | `20` |
| **sys_db_polling_concurrency** | Max concurrent DB-backed polling reads (`get_result`, `recv`, `get_event`, `read_stream`) so they can't starve the pool; non-positive disables | Half of pool size |
| **db_engine_kwargs** | Extra kwargs for SQLAlchemy `create_engine` | `None` |
| **dbos_system_schema** | Postgres schema for DBOS system tables | `"dbos"` |
| **system_database_engine** | Custom SQLAlchemy engine (skips engine creation) | `None` |
| **use_listen_notify** | Use Postgres LISTEN/NOTIFY vs polling (ignored on SQLite). Do not change after the system DB is created | `True` |
| **run_migrations** | Create/migrate the system DB on launch. Set `False` for roles that can't run DDL (migrate out of band with `dbos migrate`); launch then only verifies the schema | `True` |
| **notification_listener_polling_interval_sec** | Polling interval when polling (min `0.001`); also default `read_stream` polling interval | `1.0` |
| **notification_coalesce_sec** | Batching interval for LISTEN/NOTIFY wakeups of event/stream readers (min `0.001`) | `0.01` |
| **observability_query_timeout_sec** | Statement timeout for `list_workflows`/`list_workflow_steps`-style queries on Postgres (raises `DBOSQueryTimeoutError`); <= 0 disables | `30` |
| **conductor_key** | API key for DBOS Conductor | `None` |
| **conductor_url** | Conductor service URL (only for self-hosted) | `None` |
| **conductor_executor_metadata** | JSON dict of metadata sent to Conductor (region, instance type, etc.) | `None` |
| **conductor_metadata_only_mode** | Send only workflow metadata (never inputs/outputs/events/etc.) to Conductor | `False` |
| **enable_otlp** | Enable OpenTelemetry spans for workflows and steps | `False` |
| **otlp_traces_endpoints** | OTLP trace receiver URLs (built-in exporter) | `None` |
| **otlp_logs_endpoints** | OTLP log receiver URLs (built-in exporter) | `None` |
| **otlp_attributes** | Key-value pairs applied to all OTLP exports | `None` |
| **otel_attribute_format** | `"legacy"` (camelCase) or `"semconv"` (`dbos.*` namespace) | `"legacy"` |
| **log_level** | DBOS logger severity | `"INFO"` |
| **otlp_log_level** | OTLP-specific log level (>= `log_level`) | `log_level` |
| **console_log_level** | Console-specific log level (>= `log_level`) | `log_level` |
| **max_executor_threads** | Max threads for sync workflow/step execution | Unbounded |
| **scheduler_polling_interval_sec** | Scheduler polling interval for new schedules | `30.0` |
| **kafka_queue_polling_interval_sec** | Polling interval of the internal Kafka consumer queues (min `0.001`) | `1.0` |
| **serializer** | Custom serializer for system database | Default (pickle) |

The `application_database_url` and `database_url` fields were removed in 3.0.

### Application Name

- Must be 3-256 characters: lowercase letters, numbers, dashes, and underscores only.
- The name is the **ownership key** in the system database: workflows, queues, schedules, and application versions belong to the application that created them, and an application only runs its own workflows. Applications sharing a system database must have distinct names.
- Renaming an application requires transferring ownership of its data; stop it and transfer ownership with `dbos rename-application --from old --to new` (or `DBOSClient.rename_application`).

## Lifecycle Methods

### Listening to Specific Queues

Use `DBOS.listen_queues` after constructing `DBOS(config=...)` and **before** `DBOS.launch()` to restrict a process to dequeuing from specific queues only (useful for heterogeneous worker pools). It takes queue **names** only (not `Queue` objects) and may be called at most once:

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

`DBOS.destroy` shuts down the singleton (stops queue polling and the scheduler, closes connections) so it can be re-initialized with a new `DBOS(config=...)` - primarily used in tests.

```python
DBOS.destroy(
    workflow_completion_timeout_sec=30,   # Wait up to 30s for active workflows
    destroy_registry=False,               # Keep decorator registrations across destroy
)
```

`destroy` does not interrupt workflows that are still running after the timeout, but they can no longer checkpoint progress. Set `destroy_registry=True` only if you also want to un-register all decorated functions.

`DBOS.reset_system_database(truncate=True)` empties the DBOS system tables (much faster than the default, which drops the whole system database). It must be called **before** `DBOS.launch()` and is **destructive, test-only**.

## Least-Privilege Deployments (Migrating Out of Band)

By default, `DBOS.launch()` creates and migrates the system database, which needs DDL privileges. In production, run migrations with a privileged role and run the app with a minimal role and `run_migrations=False`.

**Incorrect (app role can't run DDL, but DBOS tries to migrate on launch):**

```python
config: DBOSConfig = {
    "name": "my-app",
    "system_database_url": os.environ["DBOS_SYSTEM_DATABASE_URL"],  # restricted role
}
```

**Correct (migrate out of band, verify on launch):**

```shell
# As a privileged user: create/upgrade DBOS tables and grant the app role access
dbos migrate -s "$ADMIN_SYSTEM_DATABASE_URL" -r my_app_role   # add --schema if not "dbos"
```

```python
config: DBOSConfig = {
    "name": "my-app",
    "application_version": "0.1.0",
    "system_database_url": os.environ["DBOS_SYSTEM_DATABASE_URL"],  # my_app_role
    "run_migrations": False,
}
```

With `run_migrations=False`, launch only verifies the schema: missing DBOS tables (or a missing SQLite file) or a schema behind this DBOS version fail launch with `DBOSInitializationError`; a missing Postgres database fails with a connection error. A schema ahead of the required version is accepted, so older processes can run beside newer peers.

If a DBA must apply the SQL, print it instead of executing (`--print-migrations` is Postgres only; its output contains `CREATE/DROP INDEX CONCURRENTLY`, so run it outside a transaction block):

```shell
dbos migrate --print-migrations all -s "$DBOS_SYSTEM_DATABASE_URL" > migrations.sql  # or a number to upgrade from
dbos migrate --print-user-role -r my_app_role -s "$DBOS_SYSTEM_DATABASE_URL" > grants.sql
```

With `run_migrations=False`, a schema behind the running DBOS version fails launch.

## Connection Poolers (PgBouncer, PlanetScale, Supabase, Neon)

When connecting through a connection pooler in **transaction mode**, set `use_listen_notify` to `False`:

```python
config: DBOSConfig = {
    "name": "my-app",
    "application_version": "0.1.0",
    "system_database_url": os.environ.get("DBOS_SYSTEM_DATABASE_URL"),
    "use_listen_notify": False,
}
```

**Why:** Postgres LISTEN is connection-scoped state — the registration lives in the backend process's memory and is tied to the TCP connection. Transaction-mode poolers (PgBouncer, PlanetScale, Supabase Supavisor, Neon) return server connections to the pool after each transaction, orphaning the LISTEN registration. Subsequent NOTIFY messages are delivered to the server connection, but the pooler has no client mapped to forward them to — so notifications are **silently discarded**.

**Symptom:** `DBOS.recv()` and `DBOS.get_event()` block indefinitely with no errors.

**Fallback behavior:** With `use_listen_notify: False`, DBOS polls the `dbos.notifications` table every 1 second (configurable via `notification_listener_polling_interval_sec`). This adds up to 1 second of latency to message/event delivery but has negligible impact on database load since the query hits an indexed lookup.

**Session-mode poolers** (PgBouncer in session mode) maintain a 1:1 client-to-server mapping for the connection lifetime, so LISTEN/NOTIFY works normally. Only transaction-mode and statement-mode poolers require this setting.

Reference: [DBOS Configuration](https://docs.dbos.dev/python/reference/configuration)
