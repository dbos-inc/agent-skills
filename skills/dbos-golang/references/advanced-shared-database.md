---
title: Share a System Database Between Applications
impact: LOW
impactDescription: Lets multiple applications (in any language) share one system database, isolated by application name, while deliberately calling each other's workflows
tags: advanced, application-name, shared-database, ownership, rename
---

## Share a System Database Between Applications

Multiple DBOS applications, potentially in different languages, can share a single system database (Go SDK v1.2.0+). Each application is identified by its configured `AppName` and owns everything it creates: workflows, steps, queues, schedules, and application versions. Applications are isolated by default but can interoperate by naming each other.

Ownership determines which application runs what:

- A workflow is dequeued, run, and recovered only by the application that owns it.
- A queue is polled only by the application that registered it, even if another application enqueues workflows on it.
- A schedule is fired only by the application that created it, and its workflows are owned by that application.
- Application versions are tracked per application, so one application's deployments do not affect which version its peers consider latest.

Queue, schedule, and version names remain globally unique across the shared database; registering a name another application owns returns an error. Workflow IDs are also global, so ID-addressed operations (`RetrieveWorkflow`, `Send`, `GetEvent`, `ReadStream`, ...) work across applications regardless of ownership. Listing operations (`ListWorkflows`, `ListQueues`, `ListSchedules`, aggregates) default to the calling application's rows.

### Calling another application's workflows

**Incorrect (enqueueing a foreign workflow without naming its owner):**

```go
// The enqueueing application owns this workflow, but it has no
// "process_order" registered — the workflow is never dequeued.
handle, _ := dbos.Enqueue[any](ctx, "orders", "process_order", "order-123")
```

**Correct (naming the owning application):**

```go
handle, err := dbos.Enqueue[any](ctx, "orders", "process_order",
    "order-123",
    // The application that implements process_order owns, dequeues, and runs it
    dbos.WithEnqueueApplicationName("order-service"),
)
result, err := handle.GetResult() // workflow IDs are global, so waiting works
```

The workflow runs on the owning application's latest registered version (leave `WithEnqueueApplicationVersion` unset). If the applications are written in different languages, pass a `dbos.PortableWorkflowArgs` as the input so the target can decode the arguments — see [advanced-interops.md](advanced-interops.md).

### Clients must name their application

A standalone client with no `AppName` sees every application's rows, but everything it creates is owned by **no** application. Unowned rows are treated as everyone's: any application may dequeue an unowned workflow, and every application fires unowned schedules and polls unowned queues. Always set `AppName` on clients when the system database is shared:

```go
client, err := dbos.NewClient(context.Background(), dbos.ClientConfig{
    DatabaseURL: os.Getenv("DBOS_SYSTEM_DATABASE_URL"),
    AppName:     "order-service", // act on behalf of this application
})
```

### Per-application options

- `dbos.WithEnqueueApplicationName(name)` — enqueue a workflow owned (and run) by another application
- `dbos.WithQueueApplicationName(name)` on `RegisterQueue` — register a queue owned by the named application (defaults to the caller's own; registering a queue owned by a different application fails)
- `ScheduleSpec.ApplicationName` — the application that owns a schedule and runs its workflows
- Listing filters: `dbos.WithFilterApplicationName(names...)` (`ListWorkflows`), `dbos.WithListQueuesApplicationNames(names...)` (`ListQueues`), `dbos.WithScheduleApplicationNames(names...)` (`ListSchedules`) — each lists the named applications' rows; rows owned by no application are always included

### Renaming an application

Ownership is recorded under the application's name, so renaming requires transferring ownership of its rows. Stop the application first (a running application would race the rename, creating new work under its old name), then:

```go
counts, err := dbos.RenameApplication(client, dbos.RenameApplicationInput{
    OldName: "old-name",
    NewName: "new-name",
    // AdoptUnclaimedRows: true, // also take rows owned by no application
})
// counts reports rows transferred, by table: Queues, Schedules, Versions, Workflows, Steps
```

Or with the CLI: `dbos rename-application --from old-name --to new-name`. The operation is idempotent — if interrupted, re-running resumes where it left off. Before adding a second application to an existing system database, adopt the pre-existing unowned rows into the first application: `dbos rename-application --to my-app --adopt-unclaimed-rows`.

Reference: [Sharing a System Database](https://docs.dbos.dev/explanations/sharing-a-system-database)
