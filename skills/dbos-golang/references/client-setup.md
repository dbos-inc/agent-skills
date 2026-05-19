---
title: Initialize Client for External Access
impact: HIGH
impactDescription: Enables external applications to interact with DBOS workflows
tags: client, external, setup, initialization
---

## Initialize Client for External Access

Use `dbos.NewClient` to interact with DBOS from external applications like API servers, CLI tools, or separate services. The Client connects directly to the DBOS system database.

**Incorrect (using full DBOS context from an external app):**

```go
// Full DBOS context requires Launch() - too heavy for external clients
ctx, _ := dbos.NewDBOSContext(context.Background(), config)
dbos.Launch(ctx)
```

**Correct (using Client):**

```go
client, err := dbos.NewClient(context.Background(), dbos.ClientConfig{
	DatabaseURL: os.Getenv("DBOS_SYSTEM_DATABASE_URL"),
})
if err != nil {
	log.Fatal(err)
}
defer client.Shutdown(10 * time.Second)

// Send a message to a workflow
err = client.Send(workflowID, "notification", "topic")

// Get an event from a workflow
event, err := client.GetEvent(workflowID, "status", 60*time.Second)

// Retrieve a workflow handle
handle, err := client.RetrieveWorkflow(workflowID)
result, err := handle.GetResult()

// List workflows
workflows, err := client.ListWorkflows(
	dbos.WithStatus([]dbos.WorkflowStatusType{dbos.WorkflowStatusError}),
)

// Workflow management
err = client.CancelWorkflow(workflowID)
err = client.CancelWorkflows([]string{"wf-1", "wf-2"})       // Bulk
handle, err = client.ResumeWorkflow(workflowID)
handles, err := client.ResumeWorkflows([]string{"wf-1", "wf-2"},
    dbos.WithResumeQueue("priority"))                          // Bulk + queue
err = client.SetWorkflowDelay(workflowID,
    dbos.WithDelayDuration(30*time.Minute))                    // Delay a queued workflow
err = client.DeleteWorkflows([]string{"wf-1"})

// Read a stream
values, closed, err := client.ClientReadStream(workflowID, "results")
ch, err := client.ClientReadStreamAsync(workflowID, "results")

// Schedule management (DB-backed schedules)
client.CreateSchedule(dbos.ClientScheduleInput{
    ScheduleName: "daily",
    WorkflowName: "dailyReport",
    Schedule:     "0 0 9 * * *",
})
client.ApplySchedules([]dbos.ClientScheduleInput{ /* ... */ })
schedules, _ := client.ListSchedules()
sched, _ := client.GetSchedule("daily")
client.PauseSchedule("daily")
client.ResumeSchedule("daily")
client.DeleteSchedule("daily")
ids, _ := client.BackfillSchedule("daily",
    time.Now().Add(-7*24*time.Hour), time.Now())
handle, _ := client.TriggerSchedule("daily")

// Application versions
versions, _ := client.ListApplicationVersions()
latest, _ := client.GetLatestApplicationVersion()
client.SetLatestApplicationVersion("v1.2.3")
```

ClientConfig options:
- `DatabaseURL` (required unless `SystemDBPool` or `SqliteSystemDB` is set): PostgreSQL/CockroachDB connection string
- `SystemDBPool`: Custom `*pgxpool.Pool` (mutually exclusive with `SqliteSystemDB`)
- `SqliteSystemDB`: Custom `*sql.DB` for SQLite
- `DatabaseSchema`: Schema name (default: `"dbos"`)
- `Logger`: Custom `*slog.Logger`
- `Serializer`: Custom `Serializer[any]` for inputs/outputs/events (defaults to JSON). The serializer must match the application that owns the workflows. See [advanced-serialization.md](advanced-serialization.md).

Always call `client.Shutdown()` when done.

Reference: [DBOS Client](https://docs.dbos.dev/golang/reference/client)
