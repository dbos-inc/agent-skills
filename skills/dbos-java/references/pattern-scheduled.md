---
title: Run Workflows on a Cron Schedule
impact: MEDIUM
impactDescription: Exactly-once scheduled execution across all application processes
tags: pattern, scheduled, cron, applySchedules, WorkflowSchedule
---

## Run Workflows on a Cron Schedule

Declare schedules with `dbos.applySchedules(...)` after launch. DBOS assigns each firing a deterministic workflow
ID derived from the schedule name and scheduled time, so every firing runs exactly once no matter how many
processes are deployed.

**Incorrect (an in-process timer):**

```java
// Fires once per process (so N times in a cluster) and loses firings on restart
Executors.newScheduledThreadPool(1)
    .scheduleAtFixedRate(() -> proxy.dailyReport(), 0, 1, TimeUnit.DAYS);
```

**Correct (declared DBOS schedules):**

```java
import dev.dbos.transact.workflow.WorkflowSchedule;

dbos.launch();

dbos.applySchedules(
    new WorkflowSchedule("every-minute", "everyMinute", "com.example.ExampleImpl", "0 * * * * *"),
    new WorkflowSchedule("daily-report", "dailyReport", "com.example.ExampleImpl", "0 0 9 * * *")
        .withCronTimezone(ZoneId.of("America/New_York"))
        .withAutomaticBackfill(true));
```

A scheduled workflow must take exactly two arguments — the scheduled fire time and the optional context object:

```java
@Workflow
public void everyMinute(Instant scheduled, Object context) {
  // scheduled: the exact cron fire time (also used to derive the workflow ID)
  // context:   the value passed via withContext(), or null
}
```

`WorkflowSchedule(scheduleName, workflowName, className, cron)` where `className` is the fully-qualified
implementation class name, or the short name set by `@WorkflowClassName`. The cron expression uses the Spring 5.3+
6-field format (`second minute hour day month weekday`). Optional settings:

- `withCronTimezone(ZoneId)` — interpret the cron in this timezone (default UTC)
- `withAutomaticBackfill(true)` — retroactively start firings missed while the app was down
- `withQueueName(String)` — enqueue firings on a specific queue instead of the default scheduler queue
- `withStatus(ScheduleStatus.PAUSED)` — create the schedule paused
- `withContext(Object)` — attach a serializable context passed to the workflow

`applySchedules` is idempotent and atomic: run it on every startup to keep code as the source of truth. It replaces
the full definition of an existing schedule (so an omitted option reverts to its default) while preserving status
and last-fired time.

Runtime management:

```java
dbos.createSchedule(new WorkflowSchedule("on-demand", "processReport", "com.example.ReportImpl", "0 0 * * * *"));
dbos.pauseSchedule("daily-report");
dbos.resumeSchedule("daily-report");
Optional<WorkflowSchedule> s = dbos.getSchedule("every-minute");
List<WorkflowSchedule> active = dbos.listSchedules(List.of(ScheduleStatus.ACTIVE), null, null);
dbos.deleteSchedule("on-demand");

// Fire now, outside the normal cadence
WorkflowHandle<?, ?> handle = dbos.triggerSchedule("daily-report");

// Retroactively run a window of missed firings
dbos.backfillSchedule("every-minute",
    Instant.parse("2026-01-01T00:00:00Z"), Instant.parse("2026-01-02T00:00:00Z"));
```

Backfills use the schedule's *current* cron expression, so widening a schedule and then backfilling generates one
execution per tick of the new expression. Tune how often the scheduler polls with
`DBOSConfig.withSchedulerPollingInterval(Duration)` (default 30 seconds).

Reference: [Scheduled Workflows](https://docs.dbos.dev/java/tutorials/scheduled-workflows)
