---
title: Use Datasource Transactions for Database Operations
impact: HIGH
impactDescription: Datasource transactions commit application writes exactly once, even across crashes and recovery
tags: step, transaction, datasource, database, postgres, sqlite, migrations
---

## Use Datasource Transactions for Database Operations

A datasource is a handle to a database you own, over which DBOS runs durable transactions. Inside a workflow, `dbos.RunAsTransaction` commits your writes and the DBOS durability record atomically, so the transaction executes exactly once. A step that writes to a database is only at-least-once: it re-executes if the process crashes after the write but before the step is checkpointed.

**Incorrect (database write in a step — may commit twice on recovery):**

```go
func checkoutWorkflow(ctx dbos.Context, item string) (int64, error) {
	return dbos.RunAsStep(ctx, func(ctx context.Context) (int64, error) {
		var id int64
		err := pool.QueryRow(ctx, "INSERT INTO orders(item) VALUES ($1) RETURNING id", item).Scan(&id)
		return id, err
	}, dbos.WithStepName("insertOrder"))
}
```

**Correct (datasource transaction):**

```go
import "github.com/jackc/pgx/v5/pgxpool"

var ds *dbos.DataSource

func main() {
	ctx, _ := dbos.NewContext(context.Background(), config)
	pool, _ := pgxpool.New(context.Background(), os.Getenv("APP_DATABASE_URL"))
	// Create every datasource before Launch: NewDataSource after Launch returns an error
	ds, _ = dbos.NewDataSource(ctx, pool, dbos.WithDataSourceName("app"))
	dbos.RegisterWorkflow(ctx, checkoutWorkflow)
	dbos.Launch(ctx)
}

func checkoutWorkflow(ctx dbos.Context, item string) (int64, error) {
	// Runs exactly once, even across crashes and recovery
	orderID, err := dbos.RunAsTransaction(ctx, ds, func(txCtx context.Context, tx dbos.Tx) (int64, error) {
		var id int64
		err := tx.QueryRow(txCtx, "INSERT INTO orders(item) VALUES ($1) RETURNING id", item).Scan(&id)
		return id, err
	}, dbos.WithStepName("insertOrder"))
	if err != nil {
		return 0, err
	}
	// Transactions and steps share the workflow's step counter and can be interleaved
	_, err = dbos.RunAsStep(ctx, func(stepCtx context.Context) (string, error) {
		return sendConfirmation(stepCtx, orderID)
	}, dbos.WithStepName("sendConfirmation"))
	return orderID, err
}
```

`dbos.NewDataSource(ctx dbos.Client, engine, opts...)` accepts a `*pgxpool.Pool` (Postgres/CockroachDB) or a `*sql.DB` (SQLite). Options: `WithDataSourceName(name)` (logs only, default `"datasource"`), `WithDataSourceSchema(schema)` (schema of the durability table, default `"dbos"`; ignored by SQLite), `WithDataSourceSkipMigrations()` (see below).

Rules:
- Call `NewDataSource` on a `Context` **before** `Launch()`. DBOS tracks every datasource created on the context so that `RewindWorkflow` also deletes the transaction checkpoints it holds ([workflow-control.md](workflow-control.md)). A standalone client may create datasources at any time.
- `RunAsTransaction` must be called from workflow code. Nesting it inside `RunAsStep` or inside another `RunAsTransaction` returns an error.
- The function receives a driver-agnostic `dbos.Tx` with `Exec`, `Query`, and `QueryRow`. DBOS commits when the function returns `nil` and rolls back when it returns an error; never call `Commit` or `Rollback` yourself.
- `RunAsTransaction` takes the same options as `RunAsStep` (`WithStepName`, `WithStepMaxRetries`, ...) plus `WithTxIsolation(level)`: `dbos.IsoLevelReadCommitted` (default), `dbos.IsoLevelRepeatableRead`, or `dbos.IsoLevelSerializable`.
- Transient database errors (connection failures, serialization and deadlock conflicts) are retried with a fresh transaction, even when your function returns them: they do not consume the step's retries and are never checkpointed as its error (v1.7.0+). Other errors returned by your function follow the step retry policy.
- If the engine is the same handle you passed as `Config.SystemDBPool` or `Config.SQLiteSystemDB`, no durability table is created: application writes and the DBOS checkpoint commit together in a single transaction.

### Running datasource migrations separately

`NewDataSource` creates or migrates a `transaction_completion` table (and a `dbos_transaction_completion_migrations` version table) in your database, which requires DDL privileges. If the application's role must not have them, migrate with a privileged role from your deployment's migration step, then make the datasource only verify its schema:

```go
// Migration script, run with a privileged role
adminPool, _ := pgxpool.New(ctx, os.Getenv("ADMIN_DATABASE_URL"))
err := dbos.MigrateDataSource(ctx, adminPool, dbos.WithMigrateApplicationRole("app_user"))

// Application, running as app_user
ds, err := dbos.NewDataSource(dbosCtx, pool, dbos.WithDataSourceSkipMigrations())
```

`dbos.MigrateDataSource(ctx context.Context, engine, opts...)` takes `WithMigrateSchema(schema)`, `WithMigrateApplicationRole(role)` (grants the role its runtime permissions: `SELECT`, `INSERT`, `DELETE` on `transaction_completion` and `SELECT` on the version table; Postgres only), and `WithMigrateLogger(logger)`. `dbos.DataSourcePermissionStatements(schema, role)` returns the same grants as SQL for your own migration tooling. With `WithDataSourceSkipMigrations()`, `NewDataSource` fails if the schema is missing or out of date. The system database is migrated the same way with `dbos.Migrate` and `Config.SkipMigrations` ([lifecycle-config.md](lifecycle-config.md)).

To commit your own writes atomically with an enqueue or a message from outside a workflow, pass your open transaction on the system database with `WithEnqueueTransaction(tx)` or `WithSendTransaction(tx)` ([client-enqueue.md](client-enqueue.md), [comm-messages.md](comm-messages.md)).

Reference: [Transactions & Datasources](https://docs.dbos.dev/golang/tutorials/transaction-tutorial)
