---
title: Configure and Launch DBOS Properly
impact: CRITICAL
impactDescription: Application won't function without proper setup
tags: configuration, launch, setup, initialization
---

## Configure and Launch DBOS Properly

Every DBOS application must create a context, register workflows and queues, then launch before running any workflows.

**Incorrect (missing configuration or launch):**

```go
// No context or launch!
func myWorkflow(ctx dbos.DBOSContext, input string) (string, error) {
	return input, nil
}

func main() {
	// This will fail - DBOS is not initialized or launched
	dbos.RegisterWorkflow(nil, myWorkflow) // panic: ctx cannot be nil
}
```

**Correct (create context, register, launch):**

```go
func myWorkflow(ctx dbos.DBOSContext, input string) (string, error) {
	return input, nil
}

func main() {
	ctx, err := dbos.NewDBOSContext(context.Background(), dbos.Config{
		AppName:     "my-app",
		DatabaseURL: os.Getenv("DBOS_SYSTEM_DATABASE_URL"),
	})
	if err != nil {
		log.Fatal(err)
	}
	defer dbos.Shutdown(ctx, 30*time.Second)

	dbos.RegisterWorkflow(ctx, myWorkflow)

	if err := dbos.Launch(ctx); err != nil {
		log.Fatal(err)
	}

	handle, err := dbos.RunWorkflow(ctx, myWorkflow, "hello")
	if err != nil {
		log.Fatal(err)
	}
	result, err := handle.GetResult()
	fmt.Println(result) // "hello"
}
```

Config fields:
- `AppName` (required): Application identifier
- `DatabaseURL` (required unless `SystemDBPool` or `SqliteSystemDB` is set): PostgreSQL/CockroachDB connection string
- `SystemDBPool`: Custom `*pgxpool.Pool` (takes precedence over `DatabaseURL`, mutually exclusive with `SqliteSystemDB`)
- `SqliteSystemDB`: Custom `*sql.DB` for SQLite (mutually exclusive with `SystemDBPool`)
- `DatabaseSchema`: Schema name (default: `"dbos"`)
- `Logger`: Custom `*slog.Logger` (defaults to stdout)
- `AdminServer`: Enable HTTP admin server (default: `false`)
- `AdminServerPort`: Admin server port (default: `3001`)
- `ConductorURL`: DBOS Conductor service URL (optional)
- `ConductorAPIKey`: DBOS Conductor API key (optional)
- `ConductorExecutorMetadata`: `map[string]any` of metadata to identify this executor in the Conductor dashboard (must be JSON-serializable)
- `ApplicationVersion`: App version (overridden by `DBOS__APPVERSION` env var; auto-computed from binary hash if not set)
- `ExecutorID`: Executor identifier (overridden by `DBOS__VMID` env var; default: `"local"`)
- `EnablePatching`: Enable code patching system (default: `false`)
- `Serializer`: Custom `Serializer[any]` for workflow inputs/outputs/events (defaults to JSON). See [advanced-serialization.md](advanced-serialization.md).
- `SchedulerPollingInterval`: How often DB-backed schedules are reconciled (default: 30s)

### Alert Handler

Register a callback for alerts delivered from DBOS Conductor. Must be called before `Launch`:

```go
dbos.SetAlertHandler(ctx, func(name, message string, metadata map[string]string) {
    log.Printf("DBOS alert [%s]: %s metadata=%v", name, message, metadata)
})

if err := dbos.Launch(ctx); err != nil {
    log.Fatal(err)
}
```

Reference: [Integrating DBOS](https://docs.dbos.dev/golang/integrating-dbos)
