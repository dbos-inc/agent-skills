---
name: dbos-rust
description: DBOS Rust SDK for building reliable, fault-tolerant applications with durable workflows on PostgreSQL. Use this skill when writing Rust code with the dbos crate, creating workflows and steps, using queues or cron schedules, integrating DBOS with Axum, using dbos::Client from external applications, or building async Rust applications that need crash recovery and durable execution.
license: MIT
metadata:
  author: dbos
  version: "1.0.0"
  organization: DBOS
  date: November 2026
  abstract: Comprehensive guide for building fault-tolerant Rust applications with DBOS. Covers configuration and lifecycle, workflow registration and WorkflowRef handles, steps, database-backed queues, communication primitives, cron schedules, debouncing, Axum integration, dbos::Client, and cross-language interop.
---

# DBOS Rust Best Practices

Guide for building reliable, fault-tolerant async Rust applications with DBOS durable workflows.

## When to Apply

Reference these guidelines when:
- Adding `dbos` to new or existing async Rust applications
- Registering workflows (`register_workflow`) and running steps (`dbos::step`, `dbos::step_with`)
- Making application SQL writes idempotent inside steps
- Using database-backed queues for concurrency, rate limiting, partitioning, priority, or delayed execution
- Implementing workflow communication (`set_event`, `get_event`, `send`, `recv`, `write_stream`, `read_stream`)
- Configuring and launching `DBOS` instances, including with Axum HTTP servers
- Using `dbos::Client` from external Rust gateways, CLIs, or companion services
- Testing DBOS workflows and steps

## Rule Categories by Priority

| Priority | Category | Impact | Prefix |
|----------|----------|--------|--------|
| 1 | Lifecycle | CRITICAL | `lifecycle-` |
| 2 | Workflow | CRITICAL | `workflow-` |
| 3 | Step | HIGH | `step-` |
| 4 | Queue | HIGH | `queue-` |
| 5 | Communication | MEDIUM | `comm-` |
| 6 | Pattern | MEDIUM | `pattern-` |
| 7 | Testing | LOW-MEDIUM | `test-` |
| 8 | Client | MEDIUM | `client-` |
| 9 | Advanced | LOW | `advanced-` |

## Critical Rules

### Installation

Add `dbos`, `tokio`, and `serde` to your Rust project (Rust 1.97+, Edition 2024):

```bash
cargo add dbos tokio --features tokio/full serde --features serde/derive
```

Or in `Cargo.toml` (do **not** rename the `dbos` dependency key if you use `dbos::select_step!`, because its procedural macro expansion references `::dbos`):

```toml
[dependencies]
dbos = "0.6"
serde = { version = "1", features = ["derive"] }
tokio = { version = "1", features = ["full"] }
```

DBOS requires a PostgreSQL or CockroachDB system database. `Config::from_env` reads **`DBOS_DATABASE_URL`** (falling back to standard `libpq` variables such as `PGUSER` and `PGPASSWORD` for omitted credentials).

### DBOS Configuration and Launch

A DBOS Rust application MUST configure an explicit `app_version` (or set `DBOS__APPVERSION`), register all workflows **before** calling `launch()`, register database-backed queues **after** `launch()`, and await `shutdown()` or `shutdown_with()` before exiting:

```rust
use std::time::Duration;
use dbos::{Config, DBOS, ShutdownOptions};

async fn greet_workflow(name: String) -> dbos::Result<String> {
    let message = dbos::step("build_greeting", || {
        let name = name.clone();
        async move { Ok(format!("Hello, {name}!")) }
    })
    .await?;
    Ok(message)
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let dbos = DBOS::new(Config {
        app_version: Some("0.1.0".to_owned()),
        ..Config::from_env("my-app")
    });

    let greet = dbos.register_workflow("greet_workflow", greet_workflow)?;

    dbos.launch().await?;

    let greeting = greet.run("Ada".to_owned()).await?;
    println!("{greeting}");

    dbos.shutdown_with(ShutdownOptions {
        workflow_completion_timeout: Duration::from_secs(30),
    })
    .await;
    Ok(())
}
```

Unlike other DBOS SDKs, `dbos` for Rust never hashes the compiled binary to invent a default version: launching with `app_version: None` when `DBOS__APPVERSION` is unset fails immediately with `Error::Config`. Set `Some("0.1.0".to_owned())` (or `Some(env!("CARGO_PKG_VERSION").to_owned())`), and prefix version, queue, and schedule names with `app_name` when multiple applications share one system database (see `references/advanced-shared-database.md`).

### Workflow and Step Structure

Workflows take **exactly one serializable input parameter `P`** (use `_: ()` for none, or a struct/tuple for multiple fields) and **no context parameter** (DBOS tracks execution context via `tokio::task_local!`). Wrap all I/O, HTTP calls, SQL queries, clocks, and randomness inside `dbos::step` or `dbos::step_with`:

```rust
use std::time::Duration;
use dbos::StepOptions;
use serde::{Deserialize, Serialize};

#[derive(Debug, thiserror::Error, Serialize, Deserialize)]
#[error("upstream error: {0}")]
struct ApiError(String);

async fn sync_user(user_id: String) -> dbos::Result<String, ApiError> {
    let profile = dbos::step_with(
        "fetch_profile",
        StepOptions {
            max_attempts: 3,
            interval: Duration::from_secs(1),
            ..StepOptions::default()
        },
        || {
            let user_id = user_id.clone();
            async move { Ok(format!("profile:{user_id}")) }
        },
    )
    .await?;

    dbos::set_event("synced_profile", &profile).await?;
    Ok(profile)
}
```

Note that `dbos` for Rust does **not** have datasources or transactional steps yet: SQL writes inside `dbos::step` commit separately from the step checkpoint (at-least-once on mid-step crash), so make database writes idempotent using `ON CONFLICT` and `dbos::workflow_id()` / `dbos::step_id()` (see `references/step-transactions.md`).

### Key Constraints

- Workflows are ONLY durable when invoked through the `WorkflowRef` returned by `register_workflow` / `register_workflow_with` (`run`, `run_with`, `start`, `start_with`)
- Register ALL workflows before `dbos.launch()`; register queues (`register_queue`) and schedules (`apply_schedules`) AFTER `dbos.launch()`
- Never capture `DBOS` inside a registered workflow closure (causes an `Arc` cycle with the workflow registry); use ambient free functions (`dbos::step`, `dbos::set_event`, `dbos::get_event`, `dbos::send`, `dbos::recv`, `dbos::sleep`, `dbos::select_workflow!`, `dbos::join_workflows!`) inside workflows
- Do NOT start/enqueue workflows or call `dbos::set_event`, `dbos::recv`, or `dbos::patch` inside `dbos::step` (`Error::InsideStep`)
- Do NOT spawn detached tasks with `tokio::spawn` expecting workflow context, and do NOT race durable calls with `tokio::select!` (use `dbos::select_step!` or `dbos::select_workflow!` instead)
- Workflows MUST be deterministic: keep clocks, `Uuid::new_v4()`, randomness, `HashMap` iteration order, and external I/O out of workflow bodies, and use `dbos::sleep` instead of `tokio::time::sleep`
- Workflow/step inputs, outputs, and custom `DurableError` types MUST implement `serde::Serialize` + `serde::de::DeserializeOwned`
- Workflow management methods on `DBOS` and `Client` use bare verbs (`cancel`, `resume`, `fork`, `rewind`, `delete`) and `queue(name)` (not `get_queue`)

## How to Use

Read individual rule files for detailed explanations and examples:

```text
references/lifecycle-config.md
references/lifecycle-axum.md
references/workflow-registration.md
references/workflow-determinism.md
references/step-transactions.md
references/queue-concurrency.md
references/workflow-control.md
```

## References

- https://docs.dbos.dev/
- https://docs.rs/dbos
- https://github.com/dbos-inc/dbos-transact-rust
