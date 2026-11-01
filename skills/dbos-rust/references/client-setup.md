---
title: Connect External Services with dbos::Client and Scope app_name
impact: MEDIUM
impactDescription: Submits, signals, and inspects workflows without running an executor or accidentally writing unclaimed global rows
tags: client, Client, ClientConfig, app_name, close, connect
---

## Connect External Services with dbos::Client and Scope app_name

Use `dbos::Client` in API gateways, CLI tools, webhook receivers, or companion microservices that interact with a DBOS system database **without** registering workflows or hosting a background executor. Always set `ClientConfig::app_name: Some("target-app".to_owned())` when acting on behalf of a single application, and call `client.close().await` during graceful shutdown.

**Incorrect (leaving ClientConfig::app_name as None on a shared database or expecting Client::connect to migrate):**

```rust
use dbos::{Client, ClientConfig};

async fn unsafe_gateway_client() -> dbos::Result<Client> {
    // Pitfall 1: `ClientConfig::from_env()` reads ONLY `DBOS_DATABASE_URL` and leaves
    // `app_name: None`! An unscoped client writes `application_name = NULL` rows that ANY app
    // sharing the database can dequeue, and bypasses peer-app ownership guards on updates!
    // Pitfall 2: `Client::connect` NEVER creates the database or runs migrations (the target
    // `DBOS` app or `dbosctl sysdb migrate` must migrate first).
    Client::connect(ClientConfig::from_env()).await
}
```

**Correct (scoping ClientConfig::app_name, cloning Client across handlers, and awaiting close):**

```rust
use dbos::{Client, ClientConfig};

async fn build_scoped_client(target_app: &str) -> dbos::Result<Client> {
    let config = ClientConfig {
        app_name: Some(target_app.to_owned()),
        max_connections: 5,
        // Keep `false` (default) on CockroachDB, transaction poolers, or when workers disable LISTEN/NOTIFY;
        // set `true` on direct Postgres when workers also run with `use_listen_notify: true`
        use_listen_notify: true,
        ..ClientConfig::from_env()
    };

    // Eagerly connects and verifies that the `dbos` schema is migrated to the current version
    Client::connect(config).await
}

async fn run_cli_action(target_app: &str, workflow_id: &str) -> dbos::Result<()> {
    let client = build_scoped_client(target_app).await?;

    if let Some(status) = client.workflow_status(workflow_id).await? {
        println!("workflow {workflow_id} is {status:?}");
    }

    // Flushes pending notifications, stops any listener task, and closes the PgPool cleanly
    client.close().await;
    Ok(())
}
```

### `Client` Lifecycle and Scoping Rules

- **Cheap `Arc` cloning**: `Client` is `Clone + Send + Sync` (`Arc`-backed). Construct one `Client` at process startup and clone it into Axum/Tonic state rather than reconnecting per request.
- **Scoped (`Some("my-app")`) vs Unscoped (`None`)**:
  - `app_name: Some("my-app")`: Stamps `"my-app"` on enqueued workflows, queues, and schedules; scopes `list_workflows`, `list_queues`, `list_schedules`, and version calls to `"my-app"` (plus legacy `NULL` rows); and refuses modifying queues, schedules, or versions owned by another app.
  - `app_name: None`: Operator/multi-tenant inspection mode that reads across all applications in the schema.
- **`use_listen_notify` mismatch trap**: `ClientConfig::use_listen_notify` defaults to `false`. Enable it **only** when the target `DBOS` executors run against PostgreSQL with `use_listen_notify: true`; enabling it when nobody calls `pg_notify` causes `client.get_event` waits to fall back to 60-second safety polls instead of 1-second polls!

Reference: [dbos::Client](https://docs.rs/dbos/latest/dbos/struct.Client.html), [dbos::ClientConfig](https://docs.rs/dbos/latest/dbos/struct.ClientConfig.html)
