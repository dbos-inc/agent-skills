---
title: Configure and Launch DBOS Properly
impact: CRITICAL
impactDescription: An application that registers late or has no version cannot launch or recover
tags: lifecycle, config, launch, shutdown
---

## Configure and Launch DBOS Properly

Create a `DBOS` instance from a `Config`, register every workflow, then call `launch`. An application version is required: DBOS for Rust doesn't compute one, so set `config.app_version` (or `DBOS__APPVERSION`) or `launch` fails. `Config::from_env(app_name)` reads the system database URL from `DBOS_SYSTEM_DATABASE_URL`. DBOS logs through `tracing`, so install a subscriber to see its logs.

**Incorrect (registering after launch, no version):**

```rust
async fn my_workflow(_: ()) -> dbos::Result<String> {
    Ok("done".to_owned())
}

#[tokio::main]
async fn main() -> dbos::Result<()> {
    // No app_version: launch fails unless DBOS__APPVERSION is set.
    let dbos = DBOS::new(Config::from_env("my-app"));
    dbos.launch().await?;
    // Registered after launch: refused with Error::AlreadyLaunched.
    let workflow = dbos.register_workflow("my_workflow", my_workflow)?;
    workflow.run(()).await?;
    Ok(())
}
```

**Correct (register, then launch, with a version):**

```rust
async fn my_workflow(_: ()) -> dbos::Result<String> {
    Ok("done".to_owned())
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    tracing_subscriber::fmt::init();

    // Reads the system database URL from DBOS_SYSTEM_DATABASE_URL.
    let mut config = Config::from_env("my-app");
    config.app_version = Some("0.1.0".to_owned());
    let dbos = DBOS::new(config);

    // Register every workflow before launch.
    let workflow = dbos.register_workflow("my_workflow", my_workflow)?;
    dbos.launch().await?;

    let result = workflow.run(()).await?;
    println!("{result}");

    dbos.shutdown().await;
    Ok(())
}
```

When creating a new application, set `app_version` to `"0.1.0"` or `env!("CARGO_PKG_VERSION")`. When editing an existing application, leave its version alone: changing it is a deployment decision (see `advanced-application-versions.md`).

`DBOS` is cheap to clone (an `Arc` inside); clone it into application state such as an Axum router's. `launch` recovers the workflows a previous run of this executor left `PENDING`. `shutdown` stops the workflows running in the process without failing them, so they are recovered on the next launch; `shutdown_with(ShutdownOptions { workflow_completion_timeout, .. })` waits for them first.

Other `Config` fields: `executor_id` (default `DBOS__VMID`, then `"local"`), `max_connections` (10), `schema` (`"dbos"`), `migrate` (true; set false when a deployment step migrates the database), `serializer`, `use_listen_notify` (true), `listen_queues` (`None`: all queues), `polling_concurrency`, `outcome_poll_interval`, `notification_coalesce`, `idle_transaction_timeout` and `scheduler_polling_interval`.

DBOS for Rust does not support DBOS Conductor or DBOS Cloud.
