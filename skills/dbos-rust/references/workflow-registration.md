---
title: Register Workflows Before Launch and Invoke via WorkflowRef
impact: CRITICAL
impactDescription: Calling a workflow function directly bypasses checkpointing, persistence, and crash recovery
tags: workflow, registration, WorkflowRef, DurableError, register_workflow
---

## Register Workflows Before Launch and Invoke via WorkflowRef

Workflows in DBOS Rust are ordinary `async` functions or closures registered with `dbos.register_workflow` (or `register_workflow_with`) before `dbos.launch()`. Registration returns a typed `WorkflowRef<P, R, E>`, and durability applies **only** when you invoke the workflow through `WorkflowRef::run`, `run_with`, `start`, or `start_with` - calling the underlying Rust function directly executes it in memory with zero persistence.

**Incorrect (calling the async function directly instead of WorkflowRef):**

```rust
use dbos::{Config, DBOS};

async fn onboard_user(email: String) -> dbos::Result<u64> {
    let user_id = dbos::step("create_account", || async { Ok(42_u64) }).await?;
    Ok(user_id)
}

async fn handle_signup(dbos: &DBOS) -> dbos::Result<u64> {
    let _onboard = dbos.register_workflow("onboard_user", onboard_user)?;
    dbos.launch().await?;

    // Not durable! Runs as a plain async call; steps are not checkpointed
    onboard_user("ada@example.com".to_owned()).await
}
```

**Correct (registering before launch and invoking through WorkflowRef):**

```rust
use dbos::{Config, DBOS, WorkflowRef};
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct SignupInput {
    email: String,
    plan: String,
}

#[derive(Debug, thiserror::Error, Serialize, Deserialize)]
enum SignupError {
    #[error("invalid email address: {0}")]
    InvalidEmail(String),
}

async fn onboard_user(input: SignupInput) -> dbos::Result<u64, SignupError> {
    if !input.email.contains('@') {
        return Err(SignupError::InvalidEmail(input.email).into());
    }
    let user_id = dbos::step("create_account", || async { Ok(42_u64) }).await?;
    Ok(user_id)
}

async fn run_app() -> Result<(), Box<dyn std::error::Error>> {
    let dbos = DBOS::new(Config {
        app_version: Some("0.1.0".to_owned()),
        ..Config::from_env("signup-service")
    });

    let onboard: WorkflowRef<SignupInput, u64, SignupError> =
        dbos.register_workflow("onboard_user", onboard_user)?;

    dbos.launch().await?;

    // Durable execution: persisted in Postgres and recoverable across crashes
    let user_id = onboard
        .run(SignupInput {
            email: "ada@example.com".to_owned(),
            plan: "pro".to_owned(),
        })
        .await?;
    println!("created user {user_id}");

    dbos.shutdown().await;
    Ok(())
}
```

### Signature, Type Bounds, and Error Channels

- **Single input value `P`**: Every workflow takes exactly one parameter `P: Serialize + DeserializeOwned + Send + 'static`. Use `_: ()` when a workflow takes no input, and use a `#[derive(Serialize, Deserialize)]` struct or tuple `(A, B)` for multiple inputs.
- **Ambient context (no `ctx` parameter)**: DBOS propagates execution state through `tokio::task_local!`, so workflow functions take only `P` and call free functions like `dbos::step`, `dbos::sleep`, `dbos::recv`, `dbos::set_event`, and `dbos::workflow_id()`.
- **Return type `dbos::Result<R, E>`**: `dbos::Result<R, E>` aliases `std::result::Result<R, dbos::Error<E>>`, where `E` defaults to the uninhabited `dbos::EngineOnly` enum. Write `dbos::Result<R>` when your workflow emits no custom domain errors, or `dbos::Result<R, MyError>` for any `MyError` implementing `std::error::Error + Serialize + DeserializeOwned + 'static` (`DurableError`).
- **Ergonomic `?` propagation**: `Error<E>` implements `From<E>`, so returning `Err(MyError::Foo)?` wraps your error in `Error::Application(...)` and checkpoints it as the workflow's outcome. Engine-only calls returning `dbos::Result<T, EngineOnly>` (such as `handle.status().await` or `dbos.cancel(...)`) convert into `dbos::Result<T, MyError>` via `.map_err(dbos::Error::lift)?` or `.lift()` before `.await`.
- **Pre-launch registration**: `register_workflow` and `register_workflow_with` fail with `Error::AlreadyLaunched` once `dbos.launch()` has succeeded, and fail with `Error::AlreadyRegistered` if two workflows share the same `QualifiedWorkflowName`.

Reference: [dbos::DBOS::register_workflow](https://docs.rs/dbos/latest/dbos/struct.DBOS.html#method.register_workflow), [dbos::WorkflowRef](https://docs.rs/dbos/latest/dbos/struct.WorkflowRef.html), [dbos::Error](https://docs.rs/dbos/latest/dbos/enum.Error.html)
