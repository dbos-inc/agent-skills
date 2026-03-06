---
title: Schedule Workflows with the Schedule API
impact: MEDIUM
impactDescription: Run workflows exactly once per time interval with full runtime management
tags: scheduled, cron, recurring, timer, schedule, create_schedule, apply_schedules
---

## Schedule Workflows with the Schedule API

Use `DBOS.create_schedule` to schedule workflows on a cron interval. Schedules are stored in the database and can be created, paused, resumed, and deleted at runtime.

**Incorrect (using the deprecated `@DBOS.scheduled` decorator):**

```python
# Deprecated - do not use decorator-based scheduling
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
- Must accept two arguments: `scheduled_time` (`datetime`) and `context` (any serializable value)
- Not supported for workflows that are methods on configured instances
- `create_schedule` fails if the schedule already exists

### Static Schedules with `apply_schedules`

For a set of schedules created on program start, use `apply_schedules` to create them atomically, updating them if they already exist:

```python
DBOS.apply_schedules([
    {
        "schedule_name": "schedule-a",
        "workflow_fn": workflow_a,
        "schedule": "*/10 * * * *",
        "context": "context-a",
    },
    {
        "schedule_name": "schedule-b",
        "workflow_fn": workflow_b,
        "schedule": "0 0 * * *",
        "context": "context-b",
    },
])
```

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

### Backfilling and Triggering

Backfill missed executions (already-executed times are automatically skipped):

```python
from datetime import datetime, timezone

DBOS.backfill_schedule(
    "my-task",
    start=datetime(2025, 1, 1, tzinfo=timezone.utc),
    end=datetime(2025, 1, 2, tzinfo=timezone.utc),
)
```

Immediately trigger a schedule:

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

Reference: [Scheduling Workflows](https://docs.dbos.dev/python/tutorials/scheduled-workflows)
