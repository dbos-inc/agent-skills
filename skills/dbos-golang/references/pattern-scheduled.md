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
func dailyReport(ctx dbos.DBOSContext, input dbos.ScheduledWorkflowInput) (any, error) {
    fmt.Println("Tick at", input.ScheduledTime, "context", input.Context)
    _, err := dbos.RunAsStep(ctx, func(ctx context.Context) (string, error) {
        return generateReport()
    }, dbos.WithStepName("generateReport"))
    return "report generated", err
}

func main() {
    ctx, _ := dbos.NewDBOSContext(context.Background(), config)
    defer dbos.Shutdown(ctx, 30*time.Second)

    dbos.RegisterWorkflow(ctx, dailyReport)
    dbos.Launch(ctx)

    err := dbos.CreateSchedule(ctx, dailyReport, dbos.CreateScheduleRequest{
        ScheduleName: "daily-report",
        Schedule:     "0 0 9 * * *", // 9 AM daily
    },
        dbos.WithScheduleContext(map[string]string{"region": "us-west"}),
        dbos.WithCronTimezone("America/Los_Angeles"),
        dbos.WithAutomaticBackfill(true),
        dbos.WithScheduleQueueName("scheduled"),
    )
    if err != nil {
        log.Fatal(err)
    }
    select {} // Block forever
}
```

Scheduled workflow functions must conform to `ScheduledWorkflowFunc`: they take a `DBOSContext` and a `ScheduledWorkflowInput` whose `ScheduledTime` is the cron tick time and whose `Context` carries the user-defined value attached to the schedule.

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
dbos.ApplySchedules(ctx, []dbos.ApplySchedulesRequest{{
    ScheduleName: "daily-report",
    WorkflowFn:   dailyReport,
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

// Trigger immediately (returns a handle to the enqueued workflow)
handle, _ := dbos.TriggerSchedule(ctx, "daily-report")

// Backfill historical ticks (returns enqueued workflow IDs)
ids, _ := dbos.BackfillSchedule(ctx, "daily-report",
    time.Now().Add(-7*24*time.Hour), time.Now())
```

The reconciler polls the DB every `Config.SchedulerPollingInterval` (default 30s) to install or remove schedule entries — useful for multi-executor deployments where one node can create a schedule that another node picks up. Each `dbos.Config` may set this interval.

`Client.CreateSchedule` / `Client.ApplySchedules` use a `ClientScheduleInput` struct (workflow referenced by name) for external creation. See [client-setup.md](client-setup.md).

Reference: [Scheduled Workflows](https://docs.dbos.dev/golang/tutorials/workflow-tutorial#scheduled-workflows)
