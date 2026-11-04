---
name: dbos-rust
description: DBOS Rust SDK (the dbos crate) for building reliable, fault-tolerant applications with durable workflows. Use this skill when writing Rust code with DBOS, creating workflows and steps, using queues, using the DBOS Client from external applications, or building Rust applications that need to be resilient to failures.
license: MIT
metadata:
  author: dbos
  version: "0.6.0"
  organization: DBOS
  date: November 2026
  abstract: Comprehensive guide for building fault-tolerant Rust applications with DBOS. Covers workflows, steps, queues, communication patterns, the determinism rules async Rust makes easy to break, and best practices for durable execution.
---

# DBOS Rust Best Practices

Guide for building reliable, fault-tolerant Rust applications with DBOS durable workflows.

## When to Apply

Reference these guidelines when:
- Adding DBOS to existing Rust code
- Creating workflows and steps
- Running DBOS calls concurrently inside a workflow
- Using queues for concurrency control
- Implementing workflow communication (events, messages, streams)
- Configuring and launching DBOS applications
- Using the DBOS Client from external applications
- Testing DBOS applications

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

Add the `dbos` crate (0.6) and Tokio:

```bash
cargo add dbos
cargo add tokio --features full
cargo add serde --features derive
cargo add thiserror tracing-subscriber
```

`serde` is for workflow inputs and outputs, `thiserror` for workflow error types, and `tracing-subscriber` to see DBOS's logs.

DBOS runs on Tokio by default; smol and async-global-executor are Cargo features (see `references/lifecycle-runtimes.md`). DBOS for Rust does not support DBOS Conductor or DBOS Cloud.

### DBOS Configuration and Launch

A DBOS application MUST create a `DBOS` instance, register every workflow, and launch before running any workflows:

```rust
use dbos::{Config, DBOS};

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

An application version is REQUIRED: DBOS for Rust doesn't compute one, so set `config.app_version` (or `DBOS__APPVERSION`) or `launch` fails. When creating a new application, set it to `"0.1.0"` or `env!("CARGO_PKG_VERSION")`. When editing an existing application, leave its configured version alone: changing it is a deployment decision (see `references/advanced-application-versions.md`).

`Config::from_env(app_name)` reads the system database URL from `DBOS_SYSTEM_DATABASE_URL`; DBOS creates the database if it doesn't exist. See `references/lifecycle-config.md` for all configuration options.

`DBOS` is cheap to clone. Outside a workflow, call its methods (`dbos.send`, `dbos.cancel`, `dbos.list_workflows`, queue and schedule management, ...); from another application, use a `Client` (see `references/client-setup.md`).

### Workflow and Step Structure

A workflow is an `async` function of one input that returns a `dbos::Result`. Any function performing complex operations or accessing external services must run as a step with `dbos::step`:

```rust
async fn generate_random_number(n: u32) -> dbos::Result<u32> {
    Ok(rand::random_range(0..n))
}

async fn workflow_function(n: u32) -> dbos::Result<u32> {
    // The closure passes arguments into the step function.
    let random_number = step("generate_random_number", || generate_random_number(n)).await?;
    Ok(random_number)
}
```

Inputs, outputs and error types must implement `serde::Serialize` and `serde::Deserialize`; an application error type must also implement `std::error::Error` (see `references/workflow-errors.md`).

### Key Constraints

- Workflows MUST be deterministic: non-deterministic operations go in steps
- Do NOT race, time out, spawn, or consume in completion order any DBOS call in a workflow body: no `tokio::select!`, `tokio::time::timeout`, `FuturesUnordered`, `buffer_unordered` or `tokio::spawn` around DBOS calls. Use `dbos::select_step!` to race, `tokio::join!` / `futures::future::try_join_all` to run concurrently, and `dbos::sleep` to wait (see `references/workflow-concurrency.md`)
- Every DBOS call takes its step ID where it is built, not where it is awaited; `dbos::patch`, `dbos::deprecate_patch` and a debouncer's `debounce` are the exceptions, so await those on their own, never inside a join
- A workflow's future must be `Send + 'static`: don't hold an `Rc`, a `std::sync::MutexGuard` or a borrow across an `.await` in a workflow body
- To start a child workflow, capture its `WorkflowRef` in the parent's registered closure; a `WorkflowRef` can't be a workflow's input
- Do NOT start or enqueue workflows, or call `dbos::recv`, `dbos::set_event` or `dbos::patch`, from within steps (`Error::InsideStep`)
- Do NOT capture the `DBOS` instance in a workflow closure: use the free functions (`dbos::send`, `dbos::cancel_workflow`, ...), which need none (see `references/workflow-capture.md`)
- All workflows MUST be registered before `launch`; queues are registered and schedules created after it
- Do NOT modify global variables from workflows or steps
- Starting a workflow with an ID already in use returns the existing workflow by default; set `workflow_id_reuse_policy: WorkflowIdReusePolicy::Reject` to get `dbos::sysdb::Error::WorkflowIdInUse` instead (see `references/pattern-idempotency.md`)
- A panic in a workflow does not crash the process: the workflow stays `PENDING`, like a crash, and is recovered on the next launch
- Steps are at-least-once and there is no transactional step: make database writes in steps idempotent

## How to Use

Read individual rule files for detailed explanations and examples:

```
references/lifecycle-config.md
references/workflow-determinism.md
references/workflow-concurrency.md
references/queue-concurrency.md
references/workflow-control.md
```

## References

- https://docs.dbos.dev/
- https://docs.rs/dbos
- https://github.com/dbos-inc/dbos-transact-rust
