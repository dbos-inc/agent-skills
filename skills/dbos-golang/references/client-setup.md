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
ctx, _ := dbos.NewContext(context.Background(), config)
dbos.Launch(ctx)
```

**Correct (using Client):**

The package-level functions that don't require a launched runtime take a `dbos.Client`, so the same APIs work with a client or a full context:

```go
client, err := dbos.NewClient(context.Background(), dbos.ClientConfig{
	DatabaseURL: os.Getenv("DBOS_SYSTEM_DATABASE_URL"),
})
if err != nil {
	log.Fatal(err)
}
defer dbos.Shutdown(client, 10*time.Second)

// Send a message to a workflow
err = dbos.Send(client, workflowID, "notification", "topic")

// Get an event from a workflow
event, err := dbos.GetEvent[string](client, workflowID, "status", 60*time.Second)

// Retrieve a workflow handle
handle, err := dbos.RetrieveWorkflow[string](client, workflowID)
result, err := handle.GetResult()

// List workflows
workflows, err := dbos.ListWorkflows(client,
	dbos.WithFilterStatus(dbos.WorkflowStatusError),
)

// Workflow management
err = dbos.CancelWorkflow(client, workflowID)
err = dbos.CancelWorkflows(client, []string{"wf-1", "wf-2"})   // Bulk
handle, err = dbos.ResumeWorkflow[string](client, workflowID)
handles, err := dbos.ResumeWorkflows[string](client, []string{"wf-1", "wf-2"},
    dbos.WithResumeQueue("priority"))                          // Bulk + queue
err = dbos.SetWorkflowDelay(client, workflowID,
    dbos.WithDelayDuration(30*time.Minute))                    // Delay a queued workflow
err = dbos.DeleteWorkflows(client, []string{"wf-1"})

// Read a stream
values, closed, err := dbos.ReadStream[string](client, workflowID, "results")
ch, err := dbos.ReadStreamAsync[string](client, workflowID, "results")

// Schedule management (DB-backed schedules)
dbos.CreateSchedule(client, dbos.ScheduleSpec{
    ScheduleName: "daily",
    WorkflowName: "dailyReport",
    Schedule:     "0 0 9 * * *",
})
dbos.ApplySchedules(client, []dbos.ScheduleSpec{ /* ... */ })
schedules, _ := dbos.ListSchedules(client)
sched, _ := dbos.GetSchedule(client, "daily")
dbos.PauseSchedule(client, "daily")
dbos.ResumeSchedule(client, "daily")
dbos.DeleteSchedule(client, "daily")
ids, _ := dbos.BackfillSchedule(client, "daily",
    time.Now().Add(-7*24*time.Hour), time.Now())
schedHandle, _ := dbos.TriggerSchedule[any](client, "daily")

// Application versions
versions, _ := dbos.ListApplicationVersions(client)
latest, _ := dbos.GetLatestApplicationVersion(client)
dbos.SetLatestApplicationVersion(client, "v1.2.3")
```

ClientConfig options:
- `DatabaseURL` (required unless `SystemDBPool` or `SQLiteSystemDB` is set): PostgreSQL/CockroachDB connection string, or a SQLite URL (`sqlite:/path/to.db`, `sqlite::memory:`; requires the blank import `_ "github.com/dbos-inc/dbos-transact-golang/dbos/driver/sqlite"`)
- `SystemDBPool`: Custom `*pgxpool.Pool` (mutually exclusive with `SQLiteSystemDB`)
- `SQLiteSystemDB`: Custom `*sql.DB` for SQLite (requires the sqlite driver import)
- `DatabaseSchema`: Schema name (default: `"dbos"`)
- `Logger`: Custom `*slog.Logger`
- `Serializer`: Custom `Serializer[any]` for inputs/outputs/events (defaults to JSON). The serializer must match the application that owns the workflows. See [advanced-serialization.md](advanced-serialization.md).
- `SystemDBStartupTimeout`: Maximum time for system-database connection and migrations (default: 2m)

Always call `dbos.Shutdown(client, timeout)` when done.

Reference: [DBOS Client](https://docs.dbos.dev/golang/reference/client)
