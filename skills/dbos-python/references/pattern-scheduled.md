---
title: Schedule Workflows with the Schedule API
impact: MEDIUM
impactDescription: Run workflows exactly once per time interval with full runtime management
tags: scheduled, cron, recurring, timer, schedule, create_schedule, apply_schedules, cron_timezone, automatic_backfill
---

## Schedule Workflows with the Schedule API

Use `DBOS.create_schedule` to schedule workflows on a cron interval. Schedules are stored in the database and can be created, paused, resumed, and deleted at runtime.

**Incorrect (`@DBOS.scheduled` decorator, removed in 3.0):**

```python
# Removed in 3.0: DBOS.scheduled no longer exists (AttributeError).
# The (scheduled_time, actual_time) signature is also gone.
@DBOS.scheduled("* * * * *")
@DBOS.workflow()
def run_every_minute(scheduled_time, actual_time):
    do_task()
```

**Correct (using `DBOS.apply_schedules`):**

```python
from datetime import datetime
from typing import Any
from dbos import DBOS

@DBOS.workflow()
def run_every_minute(scheduled_time: datetime, context: Any):
    do_task()

if __name__ == "__main__":
    DBOS(config=config)
    DBOS.launch()

    # apply_schedules is idempotent - safe to call on every restart
    DBOS.apply_schedules([{
        "schedule_name": "my-task",
        "workflow_fn": run_every_minute,
        "schedule": "* * * * *",
        "context": None,
    }])
```

Scheduled workflow requirements:
- Must accept two arguments: `scheduled_time` (`datetime`) and `context` (the schedule's context, any serializable value). Sync or `async def` workflows both work
- Not supported for workflows that are methods on configured instances; use plain functions, `@staticmethod`, or `@classmethod`
- Schedules live in the system database: `create_schedule`, `apply_schedules`, and the other schedule methods must be called **after** `DBOS.launch()` (they raise otherwise)
- `create_schedule` fails if the schedule name already exists (names are unique across all applications sharing the system database); use `apply_schedules` for idempotent setup
- `apply_schedules` upserts by name and **replaces the whole definition**: any optional field you omit (e.g. `queue_name`) is cleared. Status and last-fired time are preserved
- Schedules are owned by the application that creates them; scheduled workflows are enqueued to the owning application's latest version
- A schedule you stop applying keeps running until you `DBOS.delete_schedule` it
- Each fired workflow is tagged with the schedule name: `DBOS.list_workflows(schedule_name="my-task")` returns all runs

### `create_schedule` Parameters

```python
DBOS.create_schedule(
    schedule_name="my-task",
    workflow_fn=my_periodic_task,
    schedule="*/5 * * * *",
    context="my context",
    cron_timezone="America/New_York",   # Optional - IANA tz; defaults to UTC
    automatic_backfill=True,            # Optional - auto-backfill missed runs on startup
    queue_name="scheduled_queue",       # Optional - enqueue on a declared queue
)
```

`apply_schedules` accepts a list of dicts with the same fields.

### Routing Scheduled Workflows to a Queue

By default, scheduled workflows run on an internal queue. Set `queue_name` to enforce concurrency or rate limits:

```python
# After DBOS.launch()
DBOS.register_queue("scheduled_queue", global_concurrency=1)

DBOS.create_schedule(
    schedule_name="my-task",
    workflow_fn=my_periodic_task,
    schedule="*/5 * * * *",
    queue_name="scheduled_queue",
)
```

### Cron Timezone

Cron expressions are evaluated in UTC by default. Set `cron_timezone` to an IANA timezone (e.g. `"America/New_York"`) to evaluate in local time, handling DST correctly:

```python
DBOS.create_schedule(
    schedule_name="daily-9am-ny",
    workflow_fn=daily_task,
    schedule="0 9 * * *",
    cron_timezone="America/New_York",
)
```

### Automatic Backfill

Set `automatic_backfill=True` so missed executions are re-run on startup or when a paused schedule resumes. Otherwise, use `DBOS.backfill_schedule` manually (see below).

### Dynamic Per-Entity Schedules

Create many schedules for the same workflow, using context to differentiate:

```python
def on_customer_registration(customer_id: str):
    DBOS.create_schedule(
        schedule_name=f"customer-{customer_id}-sync",
        workflow_fn=customer_workflow,
        schedule="0 * * * *",
        context=customer_id,
    )
```

### Managing Schedules at Runtime

```python
DBOS.pause_schedule("my-task")        # Stop firing
DBOS.resume_schedule("my-task")       # Resume firing
DBOS.delete_schedule("my-task")       # Remove entirely

schedules = DBOS.list_schedules(status="ACTIVE")
schedule = DBOS.get_schedule("my-task")
```

`list_schedules` and `get_schedule` return `WorkflowSchedule` dicts with fields: `schedule_id`, `schedule_name`, `workflow_name`, `workflow_class_name`, `schedule`, `status` (`"ACTIVE"` or `"PAUSED"`), `context`, `last_fired_at`, `automatic_backfill`, `cron_timezone`, `queue_name`.

### Manual Backfill and Trigger

Backfill missed executions between two timestamps (already-executed times are skipped). Backfill uses the schedule's **current** cron expression. Both backfill and trigger enqueue on the schedule's `queue_name` (or the internal queue if none):

```python
from datetime import datetime, timezone

DBOS.backfill_schedule(
    "my-task",
    start=datetime(2025, 1, 1, tzinfo=timezone.utc),
    end=datetime(2025, 1, 2, tzinfo=timezone.utc),
)
```

Immediately trigger a schedule once:

```python
handle = DBOS.trigger_schedule("my-task")
```

### Crontab Format

```
 ┌────────────── second (optional)
 │ ┌──────────── minute
 │ │ ┌────────── hour
 │ │ │ ┌──────── day of month
 │ │ │ │ ┌────── month
 │ │ │ │ │ ┌──── day of week
 * * * * * *
```

Common patterns: `* * * * *` (every minute), `0 * * * *` (hourly), `0 0 * * *` (daily), `0 0 * * 0` (weekly Sunday).

### Managing Schedules from Another Application

Use `DBOSClient` to create/manage schedules from outside the DBOS application. The client takes a `workflow_name` string instead of a function reference. Set `application_name` on the client (or pass it to `create_schedule`) so the right application owns and runs the schedule:

```python
client.create_schedule(
    schedule_name="my-task",
    workflow_name="my_periodic_task",
    schedule="*/5 * * * *",
    context="my context",
)
```

Reference: [Scheduling Workflows](https://docs.dbos.dev/python/tutorials/scheduled-workflows)
