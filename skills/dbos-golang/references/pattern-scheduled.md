---
title: Create Scheduled Workflows
impact: MEDIUM
impactDescription: Enables durable recurring tasks managed in the database with exactly-once-per-interval guarantees
tags: pattern, scheduled, cron, recurring
---

## Create Scheduled Workflows

Use DBOS database-backed schedules to run a workflow on a cron schedule. Each schedule is persisted in Postgres, so it survives restarts, can be paused/resumed/deleted at runtime, and is picked up by any executor connected to the same database. Each tick runs exactly once per interval.

**Incorrect (manual scheduling with a goroutine):**

```go
// Not durable: missed intervals during downtime, no coordination across executors
go func() {
    for {
        generateReport()
        time.Sleep(60 * time.Second)
    }
}()
```

**Correct (DB-backed schedule):**

```go
// DB-backed scheduled workflows must use the ScheduledWorkflowFunc signature
func dailyReport(ctx dbos.Context, input dbos.ScheduledWorkflowInput) (any, error) {
    cfg, _ := dbos.DecodeScheduleContext[map[string]string](input)
    fmt.Println("Tick at", input.ScheduledTime, "region", cfg["region"])
    _, err := dbos.RunAsStep(ctx, func(ctx context.Context) (string, error) {
        return generateReport()
    }, dbos.WithStepName("generateReport"))
    return "report generated", err
}

func main() {
    ctx, _ := dbos.NewContext(context.Background(), config)
    defer dbos.Shutdown(ctx, 30*time.Second)

    dbos.RegisterWorkflow(ctx, dailyReport)
    dbos.Launch(ctx)

    err := dbos.CreateSchedule(ctx, dbos.ScheduleSpec{
        ScheduleName:      "daily-report",
        Schedule:          "0 0 9 * * *", // 9 AM daily
        Workflow:          dailyReport,
        Context:           map[string]string{"region": "us-west"},
        CronTimezone:      "America/Los_Angeles",
        AutomaticBackfill: true,
        QueueName:         "scheduled",
    })
    if err != nil {
        log.Fatal(err)
    }
    select {} // Block forever
}
```

Scheduled workflow functions must conform to `ScheduledWorkflowFunc`: they take a `Context` and a `ScheduledWorkflowInput` whose `ScheduledTime` is the cron tick time and whose `Context` carries the user-defined value attached to the schedule (JSON-serialized; decode it with `dbos.DecodeScheduleContext[T](input)`). In `ScheduleSpec`, set `Workflow` to a registered Go function, or `WorkflowName` (plus optionally `WorkflowClassName`) to reference a workflow by name — including one owned by another process or language.

DBOS crontab uses 6 fields with second precision:
```text
┌────────────── second
│ ┌──────────── minute
│ │ ┌────────── hour
│ │ │ ┌──────── day of month
│ │ │ │ ┌────── month
│ │ │ │ │ ┌──── day of week
* * * * * *
```

### Managing schedules at runtime

```go
// Apply (create-or-update) many schedules atomically
dbos.ApplySchedules(ctx, []dbos.ScheduleSpec{{
    ScheduleName: "daily-report",
    Workflow:     dailyReport,
    Schedule:     "0 0 9 * * *",
    Context:      "ctx-value",
}})

// Inspect / filter
schedules, _ := dbos.ListSchedules(ctx,
    dbos.WithScheduleStatuses(dbos.ScheduleStatusActive),
    dbos.WithScheduleWorkflowNames("dailyReport"),
    dbos.WithScheduleNamePrefixes("daily-"))
sched, _ := dbos.GetSchedule(ctx, "daily-report")

// Pause / resume / delete
dbos.PauseSchedule(ctx, "daily-report")
dbos.ResumeSchedule(ctx, "daily-report")
dbos.DeleteSchedule(ctx, "daily-report")

// Trigger immediately (returns a typed handle to the enqueued workflow)
handle, _ := dbos.TriggerSchedule[any](ctx, "daily-report")

// Backfill historical ticks (returns enqueued workflow IDs)
ids, _ := dbos.BackfillSchedule(ctx, "daily-report",
    time.Now().Add(-7*24*time.Hour), time.Now())
```

The reconciler polls the DB every `Config.SchedulerPollingInterval` (default 30s) to install or remove schedule entries — useful for multi-executor deployments where one node can create a schedule that another node picks up. Each `dbos.Config` may set this interval.

All schedule functions take a `Client`, so they also work from an external `dbos.Client` — reference the workflow by `WorkflowName` since a function pointer is only available in-process. See [client-setup.md](client-setup.md).

Reference: [Scheduled Workflows](https://docs.dbos.dev/golang/tutorials/workflow-tutorial#scheduled-workflows)
