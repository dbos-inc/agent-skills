---
title: Integrate DBOS with Axum HTTP Servers
impact: CRITICAL
impactDescription: Prevents reference cycles, premature pool connection failures, and dropped in-flight workflows during server restarts
tags: lifecycle, axum, http, state, graceful-shutdown, web
---

## Integrate DBOS with Axum HTTP Servers

DBOS Rust has no global singleton: share `DBOS` and registered `WorkflowRef` handles across HTTP handlers via Axum's `State` extractor, launch DBOS before binding or serving requests, and wire `with_graceful_shutdown` into `dbos.shutdown_with` so restarts drain active workflows cleanly. Never capture `DBOS` inside a registered workflow closure, because `DBOS` owns the workflow registry and capturing it creates an `Arc` reference cycle.

**Incorrect (capturing DBOS in workflow closure and exiting without shutdown):**

```rust
use axum::{Router, routing::post};
use dbos::{Config, DBOS};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let dbos = DBOS::new(Config {
        app_version: Some("0.1.0".to_owned()),
        ..Config::from_env("shop-api")
    });

    // Reference cycle: DBOS -> Registry -> closure -> DBOS!
    let dbos_for_wf = dbos.clone();
    let checkout = dbos.register_workflow("checkout", move |item: String| {
        let _dbos = dbos_for_wf.clone();
        async move { Ok::<_, dbos::Error>(item) }
    })?;

    dbos.launch().await?;

    let app = Router::new().route(
        "/checkout",
        post(move || {
            let checkout = checkout.clone();
            async move { checkout.run("widget".to_owned()).await.unwrap() }
        }),
    );

    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await?;
    // Process exits abruptly on SIGINT without draining or closing DBOS pools
    axum::serve(listener, app).await?;
    Ok(())
}
```

**Correct (shared AppState, lazy app pool, pre-launch registration, graceful shutdown):**

```rust
use std::time::Duration;
use axum::{
    Json, Router,
    extract::{Path, State},
    http::StatusCode,
    response::{IntoResponse, Response},
    routing::post,
};
use dbos::{
    Config, DBOS, QueueConflict, QueueOptions, ShutdownOptions, StartOptions, WorkflowRef,
};
use serde::{Deserialize, Serialize};
use sqlx::{PgPool, postgres::PgPoolOptions};

#[derive(Debug, thiserror::Error, Serialize, Deserialize)]
#[error("database error: {0}")]
struct StoreError(String);

#[derive(Clone)]
struct AppState {
    dbos: DBOS,
    pool: PgPool,
    checkout: WorkflowRef<String, String, StoreError>,
}

async fn reserve_inventory(pool: &PgPool, sku: &str) -> dbos::Result<String, StoreError> {
    sqlx::query_scalar::<_, String>("SELECT $1")
        .bind(sku)
        .fetch_one(pool)
        .await
        .map_err(|e| StoreError(e.to_string()).into())
}

async fn checkout_workflow(pool: PgPool, sku: String) -> dbos::Result<String, StoreError> {
    dbos::step("reserve_inventory", || reserve_inventory(&pool, &sku)).await
}

struct ApiError(String);

impl<E: std::fmt::Display> From<dbos::Error<E>> for ApiError {
    fn from(err: dbos::Error<E>) -> Self {
        Self(err.to_string())
    }
}

impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        (StatusCode::INTERNAL_SERVER_ERROR, self.0).into_response()
    }
}

async fn start_checkout(
    State(state): State<AppState>,
    Path(idempotency_key): Path<String>,
    Json(sku): Json<String>,
) -> Result<StatusCode, ApiError> {
    state
        .checkout
        .start_with(
            sku,
            StartOptions {
                workflow_id: Some(&idempotency_key),
                ..StartOptions::default()
            },
        )
        .await?;
    Ok(StatusCode::ACCEPTED)
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let config = Config {
        app_version: Some(env!("CARGO_PKG_VERSION").to_owned()),
        ..Config::from_env("shop-api")
    };
    let database_url = config.database_url.clone();
    let dbos = DBOS::new(config);

    // 1. Connect lazily: dbos.launch() creates the database if it does not exist yet
    let pool = PgPoolOptions::new()
        .max_connections(10)
        .connect_lazy(&database_url)?;

    // 2. Register workflows before launch(), capturing cloned pools/dependencies (never DBOS)
    let checkout = {
        let pool = pool.clone();
        dbos.register_workflow("checkout", move |sku: String| {
            let pool = pool.clone();
            async move { checkout_workflow(pool, sku).await }
        })?
    };

    // 3. Launch DBOS, then register database-backed queues and run app DDL
    dbos.launch().await?;
    dbos.register_queue(
        "orders",
        QueueOptions {
            worker_concurrency: Some(5),
            ..QueueOptions::default()
        },
        QueueConflict::NeverUpdate,
    )
    .await?;

    // 4. Serve HTTP traffic and drain both Axum and DBOS on Ctrl-C
    let router = Router::new()
        .route("/checkout/{idempotency_key}", post(start_checkout))
        .with_state(AppState {
            dbos: dbos.clone(),
            pool,
            checkout,
        });

    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await?;
    axum::serve(listener, router)
        .with_graceful_shutdown(async {
            let _ = tokio::signal::ctrl_c().await;
        })
        .await?;

    dbos.shutdown_with(ShutdownOptions {
        workflow_completion_timeout: Duration::from_secs(30),
    })
    .await;
    Ok(())
}
```

### Key Server Integration Rules

- **`DBOS` and `WorkflowRef` are cheap to clone**: `DBOS` wraps an `Arc`, and `WorkflowRef<P, R, E>` implements `Clone` regardless of whether `P`, `R`, or `E` implements `Clone` (note that `WorkflowRef` is `Clone`, not `Copy`).
- **Use `connect_lazy` for application pools before `launch()`**: When `Config::migrate` is `true`, `dbos.launch()` creates the target database if missing, so eager `connect` calls before `launch()` fail on a fresh PostgreSQL instance.
- **Wrap driver errors in a serializable `DurableError`**: Database and HTTP client errors (`sqlx::Error`, `reqwest::Error`) do not implement `serde::Serialize` + `DeserializeOwned`, so convert them into a serializable error type at the step boundary.

Reference: [DBOS Rust Starter Demo](https://github.com/dbos-inc/dbos-transact-rust/tree/main/demo-apps/dbos-rust-starter), [DBOS Rust Widget Store Demo](https://github.com/dbos-inc/dbos-transact-rust/tree/main/demo-apps/dbos-rust-widget-store)
