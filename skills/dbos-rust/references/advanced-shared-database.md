---
title: Avoid Name Collisions When Sharing One System Database Across Apps
impact: LOW
impactDescription: Prevents RegisteredByAnother launch failures on app_version, queues, and schedules in multi-service schemas
tags: advanced, shared-database, multi-app, app_name, app_version, rename_application
---

## Avoid Name Collisions When Sharing One System Database Across Apps

Multiple DBOS applications (including polyglot services in Rust, Python, TypeScript, Go, and Java) can share a single PostgreSQL system database schema (`Config::schema`, default `"dbos"`). While DBOS isolates workflow recovery and queue workers by `app_name`, **application version names (`application_versions.version_name`), queue names (`queues.name`), and schedule names (`workflow_schedules.schedule_name`) are globally unique across the entire schema**! Prefix all three with your `app_name` whenever multiple services share one system database.

**Incorrect (two apps sharing one schema with unprefixed `"0.1.0"` versions and `"jobs"` queue):**

```rust
use dbos::{Config, DBOS, QueueConflict, QueueOptions};

async fn colliding_shared_db_apps(shared_url: &str) -> dbos::Result<()> {
    let billing = DBOS::new(Config {
        // Collision 1: `application_versions.version_name` is globally UNIQUE across all apps!
        app_version: Some("0.1.0".to_owned()),
        ..Config::new("billing-svc", shared_url)
    });
    billing.launch().await?;
    billing
        .register_queue("jobs", QueueOptions::default(), QueueConflict::NeverUpdate)
        .await?;

    let shipping = DBOS::new(Config {
        // Fails at `launch()` with `RegisteredByAnother` because `"billing-svc"` already owns `"0.1.0"`!
        app_version: Some("0.1.0".to_owned()),
        ..Config::new("shipping-svc", shared_url)
    });
    shipping.launch().await?;

    // Collision 2: Would also fail with `RegisteredByAnother` on queue `"jobs"`!
    shipping
        .register_queue("jobs", QueueOptions::default(), QueueConflict::NeverUpdate)
        .await?;
    Ok(())
}
```

**Correct (valid lowercase app_name, app-prefixed versions/queues/schedules, and rename_application):**

```rust
use dbos::{Config, DBOS, QueueConflict, QueueOptions};
use dbos::sysdb::SystemDatabase;
use dbos::sysdb::postgres::{Config as SysdbConfig, PostgresSystemDatabase};
use dbos::sysdb::types::{ApplicationRowCounts, RenameBatching, RenameFrom};

async fn launch_isolated_peer_app(app_name: &str, release: &str, shared_url: &str) -> dbos::Result<DBOS> {
    // `app_name` must be 3..=256 lowercase ASCII chars in `[a-z0-9_-]`
    let dbos = DBOS::new(Config {
        app_version: Some(format!("{app_name}-{release}")),
        ..Config::new(app_name, shared_url)
    });
    let _wf = dbos.register_workflow("process", |id: String| async move {
        Ok::<_, dbos::Error>(id)
    })?;

    dbos.launch().await?;
    dbos.register_queue(
        &format!("{app_name}-jobs"),
        QueueOptions::default(),
        QueueConflict::UpdateIfLatestVersion,
    )
    .await?;

    Ok(dbos)
}

// Renaming an existing app (or adopting legacy unclaimed `application_name IS NULL` rows)
async fn migrate_app_ownership(
    shared_url: &str,
    old_name: &str,
    new_name: &str,
) -> Result<ApplicationRowCounts, dbos::sysdb::Error> {
    let mut cfg = SysdbConfig::new(shared_url);
    cfg.migrate = false;
    let sysdb = PostgresSystemDatabase::connect(&cfg).await?;

    // Atomically moves queues, schedules, versions, and active workflows in one transaction,
    // then migrates terminal workflow history in batches of 10,000 rows
    sysdb
        .rename_application(
            RenameFrom::Application(old_name),
            new_name,
            RenameBatching::default(),
        )
        .await
}
```

### Shared System Database Checklist

- **`app_name` character rules**: Must be `3..=256` ASCII characters matching `[a-z0-9_-]` (uppercase characters, dots, or colons fail validation with `Error::Config`). Note that on DBOS Cloud (`DBOS__CLOUD="true"`), `DBOS_APP_NAME` overrides `Config::app_name`.
- **Always prefix `app_version`, queue names, and schedule names** with `{app_name}-` (unless intentionally sharing an unclaimed queue across apps). Workflow names (`register_workflow`) do **not** need prefixing because workflow execution is already scoped by `app_name` and `app_version`.
- **Always set `ClientConfig::app_name = Some(...)`**: An unscoped `Client` (`app_name: None`) writes unclaimed (`NULL`) workflows/queues/schedules that **every** application sharing the schema is eligible to claim!

Reference: [dbos::Config::app_name](https://docs.rs/dbos/latest/dbos/struct.Config.html#structfield.app_name), [dbos::sysdb::SystemDatabase::rename_application](https://docs.rs/dbos/latest/dbos/sysdb/trait.SystemDatabase.html#tymethod.rename_application), [dbos::sysdb::types::RenameFrom](https://docs.rs/dbos/latest/dbos/sysdb/types/enum.RenameFrom.html), [Sharing a System Database](https://docs.dbos.dev/explanations/sharing-a-system-database)
