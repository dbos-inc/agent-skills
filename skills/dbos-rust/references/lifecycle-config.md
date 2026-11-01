---
title: Configure, Register, and Launch DBOS Properly
impact: CRITICAL
impactDescription: Application fails to launch or skips workflow recovery without proper configuration and registration order
tags: configuration, launch, shutdown, app_version, migrations, pooler, lifecycle
---

## Configure, Register, and Launch DBOS Properly

Every DBOS Rust application builds a `Config` with an explicit application version, constructs a `DBOS` instance, registers all workflows before calling `launch()`, and awaits `shutdown()` or `shutdown_with()` before exiting. Unlike other DBOS SDKs, `dbos` never hashes the compiled binary to invent a fallback version, so launching without `Config::app_version` or `DBOS__APPVERSION` fails immediately with `Error::Config`.

**Incorrect (missing app_version, registering after launch, no shutdown):**

```rust
use dbos::{Config, DBOS};

async fn process_order(order_id: String) -> dbos::Result<String> {
    Ok(format!("processed {order_id}"))
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Fails at launch() if DBOS__APPVERSION is unset: Rust computes no binary hash
    let dbos = DBOS::new(Config::from_env("orders-app"));
    dbos.launch().await?;

    // Fails with Error::AlreadyLaunched: registry freezes at launch()
    let workflow = dbos.register_workflow("process_order", process_order)?;
    workflow.run("ord-1".to_owned()).await?;
    Ok(())
}
```

**Correct (set app_version, register before launch, drain on shutdown):**

```rust
use std::time::Duration;
use dbos::{Config, DBOS, ShutdownOptions};

async fn process_order(order_id: String) -> dbos::Result<String> {
    Ok(format!("processed {order_id}"))
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let config = Config {
        app_version: Some(env!("CARGO_PKG_VERSION").to_owned()),
        max_connections: 20,
        ..Config::from_env("orders-app")
    };
    let dbos = DBOS::new(config);

    // Register every workflow BEFORE launch()
    let workflow = dbos.register_workflow("process_order", process_order)?;

    // Connects, migrates, re-enqueues abandoned PENDING workflows, and starts background loops
    dbos.launch().await?;

    let output = workflow.run("ord-1".to_owned()).await?;
    println!("{output}");

    // Drain in-flight workflows up to 30s; anything left stays PENDING for recovery
    dbos.shutdown_with(ShutdownOptions {
        workflow_completion_timeout: Duration::from_secs(30),
    })
    .await;
    Ok(())
}
```

### Configuration Fields and Environment Precedence

`Config` is intentionally non-`#[non_exhaustive]` so you can override individual fields with struct update syntax (`Config { ..Config::from_env("my-app") }`):

- `app_name`: Required application owner key, 3 to 256 ASCII characters matching `[a-z0-9-_]`. Uppercase letters, dots, or spaces fail at `launch()` with `Error::Config`.
- `database_url`: PostgreSQL or CockroachDB connection URL. `Config::from_env` reads **`DBOS_DATABASE_URL`** (not `DATABASE_URL`). Omitted credentials fall back to standard `libpq` variables (`PGUSER`, `PGPASSWORD`, `PGOPTIONS`).
- `app_version`: Required code version (`Some("0.1.0".to_owned())` for new apps, `Some(env!("CARGO_PKG_VERSION").to_owned())` for crate releases, or `None` when supplied via `DBOS__APPVERSION`). Off DBOS Cloud, `Some(...)` overrides `DBOS__APPVERSION`; on DBOS Cloud (`DBOS__CLOUD="true"`), `DBOS_APP_NAME`, `DBOS__APPVERSION`, and `DBOS__VMID` always win.
- `executor_id`: Identifies this process (`None` falls back to `DBOS__VMID`, then `"local"`). Set a distinct `executor_id` per replica when running multiple workers.
- `max_connections`: Maximum system database pool size (default `10`; `0` is rejected).
- `schema`: System database schema name (default `"dbos"`). Must match across all applications sharing a database.
- `migrate`: When `true` (default), `launch()` creates the database if missing and applies pending schema migrations. When `false`, `launch()` verifies the schema version and fails if unmigrated or outdated.
- `use_listen_notify`: Uses PostgreSQL `LISTEN`/`NOTIFY` for low-latency event, message, and stream wake-ups (default `true`; automatically disabled on CockroachDB).
- `polling_concurrency`: Bounds concurrent polling queries (`PollingConcurrency::Default` caps at `max(pool_size / 2, 1)`, reserving half the pool for checkpoints and queue claims; also `Unlimited` or `AtMost(NonZeroU32)`).
- `listen_queues`: `None` dequeues all application queues; `Some(names)` restricts dequeuing to the listed queues; `Some(vec![])` dequeues **no** user queues (enqueue-only node). `_dbos_internal_queue` is always polled.
- `outcome_poll_interval` (default `1s`), `scheduler_polling_interval` (default `30s`), `notification_coalesce` (default `10ms`; `Some(Duration::ZERO)` disables coalescing), `idle_transaction_timeout` (`IdleTransactionTimeout::Default` sets `60s`; `ServerSetting` leaves Postgres untouched).

### Connection Poolers and Out-of-Band Migrations

When connecting through a **transaction-mode** connection pooler (PgBouncer with `pool_mode=transaction`, Supavisor, Neon pooled endpoints), set `use_listen_notify: false`. `DBOS` runs a 3-second `pg_notify` self-test on connect and falls back to 1-second polling with a warning if notifications are dropped, but disabling `use_listen_notify` upfront avoids maintaining an idle listener connection.

When the application role lacks DDL privileges, run migrations in a privileged deployment step (either `dbosctl sysdb migrate` or `dbos::sysdb::migrations::runner::run`) and launch the application with `migrate: false`:

```rust
use dbos::sysdb::migrations::runner;
use sqlx::postgres::PgPoolOptions;

async fn run_migrations(admin_url: &str) -> Result<(), Box<dyn std::error::Error>> {
    let pool = PgPoolOptions::new().max_connections(2).connect(admin_url).await?;
    runner::run(&pool, "dbos", true).await?;
    pool.close().await;
    Ok(())
}
```

### Recovery and Graceful Shutdown

`launch()` synchronously re-enqueues any `PENDING` workflows belonging to `(app_name, executor_id, app_version)` before returning, so abandoned runs resume from their last completed step across the worker fleet (up to 100 recovery attempts before moving to `MAX_RECOVERY_ATTEMPTS_EXCEEDED`).

`dbos.shutdown().await` aborts in-flight tasks immediately, while `dbos.shutdown_with(ShutdownOptions { workflow_completion_timeout })` stops queue and schedule producers first and waits up to the timeout for active workflows to finish cleanly. Workflows still running when the timeout expires are aborted in memory and left `PENDING` in PostgreSQL (never marked `ERROR`), so the next launch recovers them from their last checkpoint. Note that DBOS Conductor integration and `auto_upgrade` are not yet part of `dbos` for Rust.

Reference: [dbos::Config](https://docs.rs/dbos/latest/dbos/struct.Config.html), [dbos::DBOS](https://docs.rs/dbos/latest/dbos/struct.DBOS.html), [Workflow Recovery](https://docs.dbos.dev/production/workflow-recovery)
