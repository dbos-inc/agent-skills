---
title: Configure and Launch DBOS Properly
impact: CRITICAL
impactDescription: Application won't function without proper setup
tags: configuration, launch, setup, initialization, conductor, migrations, enterprise, auto-upgrade
---

## Configure and Launch DBOS Properly

Every DBOS application must create a context, register workflows and queues, then launch before running any workflows.

**Incorrect (missing configuration or launch):**

```go
// No context or launch!
func myWorkflow(ctx dbos.Context, input string) (string, error) {
	return input, nil
}

func main() {
	// This will fail - DBOS is not initialized or launched
	dbos.RegisterWorkflow(nil, myWorkflow) // panic: ctx cannot be nil
}
```

**Correct (create context, register, launch):**

```go
func myWorkflow(ctx dbos.Context, input string) (string, error) {
	return input, nil
}

func main() {
	ctx, err := dbos.NewContext(context.Background(), dbos.Config{
		AppName:            "my-app",
		ApplicationVersion: "0.1.0",
		DatabaseURL:        os.Getenv("DBOS_SYSTEM_DATABASE_URL"),
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
- `AppName` (required): Application identifier, 3 to 256 characters of lowercase letters, digits, dashes, and underscores. Any other name logs a warning, and fails `NewContext` when Conductor is configured. Applications [sharing a system database](advanced-shared-database.md) must each have a distinct name — the name identifies which application owns each workflow, queue, schedule, and version, and applications only run their own workflows
- `DatabaseURL` (required unless `SystemDBPool` or `SQLiteSystemDB` is set): PostgreSQL/CockroachDB connection string, or a SQLite URL (`sqlite:/path/to.db`, `sqlite:relative.db`, `sqlite::memory:`). SQLite requires the blank import `_ "github.com/dbos-inc/dbos-transact-golang/dbos/driver/sqlite"`. A `pool_max_conns` URL parameter overrides the default maximum pool size (20)
- `SystemDBPool`: Custom `*pgxpool.Pool` (takes precedence over `DatabaseURL`, mutually exclusive with `SQLiteSystemDB`)
- `SQLiteSystemDB`: Custom `*sql.DB` for SQLite (mutually exclusive with `SystemDBPool`; requires the sqlite driver import)
- `DatabaseSchema`: Schema name (default: `"dbos"`)
- `Logger`: Custom `*slog.Logger` (defaults to stdout)
- `ConductorAPIKey`: DBOS Conductor API key. Setting it requires importing `go.dbos.dev/dbos-enterprise-go` (see [Connecting to Conductor](#connecting-to-conductor))
- `ConductorURL`: DBOS Conductor service URL; only set it when self-hosting Conductor
- `ConductorExecutorMetadata`: `map[string]any` of metadata to identify this executor in the Conductor dashboard (must be JSON-serializable)
- `ConductorMetadataOnlyMode`: Send only workflow metadata to Conductor, never workflow data (inputs, outputs, errors, step outputs, events, messages, streams, or schedule context), whatever the console setting (default: `false`)
- `TransactEnterpriseKey`: DBOS key that entitles the application to Transact Enterprise features (auto-upgrade) without connecting to Conductor. For development only: in production, connect to Conductor, whose `ConductorAPIKey` stands in for it
- `AutoUpgrade`: Auto-upgrade every registered workflow except those registered with `dbos.WithAutoUpgrade(false)` (default: `false`; requires the enterprise import and an entitled key; see [advanced-patching.md](advanced-patching.md#auto-upgrade))
- `AutoUpgradeInterval`: Time between two auto-upgrade passes (default: 30s)
- `AutoUpgradeGracePeriod`: How long the latest application version must have been the latest before auto-upgrade re-enqueues `PENDING` workflows left on older versions; `ENQUEUED` and `DELAYED` workflows move without waiting (default: 0)
- `ApplicationVersion`: App version — set to `"0.1.0"` in new applications (overridden by `DBOS__APPVERSION` env var; auto-computed from binary hash if not set)
- `ExecutorID`: Executor identifier (overridden by `DBOS__VMID` env var; default: `"local"`)
- `EnablePatching`: Deprecated (v1.8.0). `dbos.Patch` and `dbos.DeprecatePatch` no longer require it; it only pins the application version to `"PATCHING_ENABLED"` and logs a warning. Set `ApplicationVersion` instead
- `Serializer`: Custom `Serializer[any]` for workflow inputs/outputs/events (defaults to JSON). See [advanced-serialization.md](advanced-serialization.md).
- `SchedulerPollingInterval`: How often DB-backed schedules are reconciled (default: 30s)
- `SystemDBStartupTimeout`: Maximum time for system-database connection and schema migration or verification (default: 2m)
- `SystemDBIdleTransactionTimeout`: Postgres `idle_in_transaction_session_timeout` for the connections DBOS creates (default: 1m; a negative duration leaves the server setting)
- `NotificationCoalesceInterval`: How often stream-write and set-event notifications are batched before being sent
- `SkipMigrations`: Verify the system database schema on startup instead of creating and migrating it (default: `false`; see [Migrating the system database separately](#migrating-the-system-database-separately))

`Launch()` starts the system database pool, the scheduler, the queue runner, a workflow timeouts monitor that durably cancels this application's workflows past their deadline, and, if configured, the Conductor connection and the auto-upgrade pass. It also recovers this executor's `PENDING` workflows, except when Conductor is configured, where Conductor drives recovery.

### Connecting to Conductor

Connecting to DBOS Conductor (setting `ConductorAPIKey`, or running in DBOS Cloud) requires importing DBOS Enterprise for Go, `go.dbos.dev/dbos-enterprise-go`, which must be at the same minor version as the core module. Add it with `go get go.dbos.dev/dbos-enterprise-go@v1.8.0` and import it once, anywhere in the program:

```go
import (
	"github.com/dbos-inc/dbos-transact-golang/dbos"
	_ "go.dbos.dev/dbos-enterprise-go" // required for Conductor and auto-upgrade
)

ctx, err := dbos.NewContext(context.Background(), dbos.Config{
	AppName:            "my-app",
	ApplicationVersion: "0.1.0",
	DatabaseURL:        os.Getenv("DBOS_SYSTEM_DATABASE_URL"),
	ConductorAPIKey:    os.Getenv("DBOS_CONDUCTOR_KEY"),
	// ConductorMetadataOnlyMode: true, // never send workflow data to Conductor
})
```

Without the import, `NewContext` fails with an initialization error whenever `ConductorAPIKey` is set or the application runs in DBOS Cloud. Conductor configuration is otherwise unchanged; applications that neither connect to Conductor nor auto-upgrade workflows need no import. The same import provides auto-upgrade, which also requires a key whose organization is entitled to it (`ConductorAPIKey`, or `TransactEnterpriseKey` in development); see [advanced-patching.md](advanced-patching.md#auto-upgrade).

### Alert Handler

Register a callback for alerts delivered from DBOS Conductor. Must be called before `Launch`, on a context connected to Conductor as above:

```go
dbos.SetAlertHandler(ctx, func(name, message string, metadata map[string]string) {
    log.Printf("DBOS alert [%s]: %s metadata=%v", name, message, metadata)
})

if err := dbos.Launch(ctx); err != nil {
    log.Fatal(err)
}
```

### Migrating the system database separately

`NewContext` creates the system database if needed and runs pending schema migrations, which requires DDL privileges. To run the application with a role that has none, migrate from your deployment's migration step with a privileged role (or the `dbos migrate` CLI command), then start the application with `SkipMigrations`, which makes `NewContext` verify the schema and fail if it is missing or behind:

```go
// Migration step, run with a privileged role
err := dbos.Migrate(ctx, os.Getenv("ADMIN_DATABASE_URL"), dbos.WithMigrateApplicationRole("app_user"))

// Application, running as app_user
ctx, err := dbos.NewContext(context.Background(), dbos.Config{
	AppName:        "my-app",
	DatabaseURL:    os.Getenv("DBOS_SYSTEM_DATABASE_URL"),
	SkipMigrations: true,
})
```

`dbos.Migrate(ctx context.Context, databaseURL string, opts ...MigrateOption)` takes `WithMigrateSchema(schema)` (default `"dbos"`), `WithMigrateApplicationRole(role)` (grants the role access to the migrated schema; Postgres only), and `WithMigrateLogger(logger)`. For your own migration tooling, `dbos.MigrationStatements(schema, from)` returns the SQL of the migrations from version `from` onwards and `dbos.PermissionStatements(schema, role)` the grants. A standalone client never migrates the system database. Datasources have their own schema, migrated with `MigrateDataSource` ([step-transactions.md](step-transactions.md)).

Reference: [Integrating DBOS](https://docs.dbos.dev/golang/integrating-dbos), [DBOS Transact Enterprise](https://docs.dbos.dev/explanations/transact-enterprise)
