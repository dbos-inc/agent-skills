---
title: Unit-Test Steps Plainly and Isolate PostgreSQL Workflow Tests
impact: LOW-MEDIUM
impactDescription: Runs step tests in milliseconds without PostgreSQL and prevents cross-test state leaks in workflow integration tests
tags: test, tokio-test, integration-test, shutdown, unit-test, postgres
---

## Unit-Test Steps Plainly and Isolate PostgreSQL Workflow Tests

DBOS Rust makes step logic directly unit-testable with **zero database setup**: calling `dbos::step` or `dbos::step_with` outside any workflow executes the closure plainly in memory (preserving retry behavior on `StepOptions`!). For full workflow integration tests (replay, queues, events, schedules), give each test an isolated PostgreSQL database or schema, set a fast `outcome_poll_interval`, and always await `dbos.shutdown().await` before tearing down the database.

**Incorrect (sharing one database across parallel workflow tests and skipping shutdown):**

```rust
use dbos::{Config, DBOS};

async fn flaky_workflow_test() -> dbos::Result<()> {
    // Sharing one `database_url` and `app_name` across parallel `#[tokio::test]` functions causes
    // one test's `launch()` recovery and queue workers to steal another test's workflows!
    let dbos = DBOS::new(Config {
        app_version: Some("test".to_owned()),
        ..Config::new("shared-app", "postgres://localhost:5432/shared_test_db")
    });
    let wf = dbos.register_workflow("add_one", |n: u32| async move { Ok::<_, dbos::Error>(n + 1) })?;
    dbos.launch().await?;

    assert_eq!(wf.run(1).await?, 2);
    // Missing `dbos.shutdown().await` leaves background listener/polling tasks hitting Postgres!
    Ok(())
}
```

**Correct (plain step unit tests plus isolated, cleanly shut down workflow integration tests):**

```rust
use std::time::Duration;
use dbos::{Config, DBOS, Enqueue, QueueConflict, QueueOptions, StartOptions};
use dbos::sysdb::types::WorkflowStatus;

async fn calculate_tax(subtotal_cents: u64) -> dbos::Result<u64> {
    dbos::step("compute_tax", || async move { Ok(subtotal_cents * 8 / 100) }).await
}

// 1. Unit test: `dbos::step` degrades to plain async execution outside a workflow (no Postgres!)
async fn unit_test_step_without_dbos() -> dbos::Result<()> {
    let tax = calculate_tax(1_000).await?;
    assert_eq!(tax, 80);
    Ok(())
}

// 2. Integration test: isolated database URL + fast poll interval + explicit `shutdown().await`
async fn integration_test_workflow_and_parked_queue(isolated_db_url: &str) -> dbos::Result<()> {
    let dbos = DBOS::new(Config {
        app_version: Some("test-v1".to_owned()),
        outcome_poll_interval: Some(Duration::from_millis(50)),
        // `Some(vec![])` parks user queues deterministically until we choose to inspect or release!
        listen_queues: Some(vec![]),
        ..Config::new("checkout-test", isolated_db_url)
    });

    let tax_wf = dbos.register_workflow("calculate_tax", calculate_tax)?;
    dbos.launch().await?;
    dbos.register_queue("orders", QueueOptions::default(), QueueConflict::AlwaysUpdate)
        .await?;

    // Direct run executes immediately
    assert_eq!(tax_wf.run(2_500).await?, 200);

    // Enqueued run stays deterministically `ENQUEUED` while `listen_queues` excludes `"orders"`
    let queued = tax_wf
        .start_with(
            5_000,
            StartOptions {
                queue: Some(Enqueue::new("orders")),
                ..StartOptions::default()
            },
        )
        .await?;
    assert_eq!(queued.status().await?, WorkflowStatus::Enqueued);

    // Always shut down before dropping the test database so background loops close cleanly
    dbos.shutdown().await;
    Ok(())
}
```

### Testing Tips for DBOS Rust

- **What works without `DBOS` vs what requires a workflow**: `dbos::step`, `dbos::step_with`, `dbos::select_step!`, and `dbos::sleep` degrade to plain execution outside a workflow. Workflow state primitives (`dbos::set_event`, `dbos::recv`, `dbos::write_stream`, `dbos::patch`) require a workflow row and return `Err(Error::NotInWorkflow)` outside `DBOS`.
- **Fast template-database cloning**: To run dozens of PostgreSQL workflow tests quickly, run `dbos::sysdb::migrations::runner::run(&pool, "dbos", true).await?` once against a template database, clone per-test databases via `CREATE DATABASE test_xyz TEMPLATE dbos_template`, and launch each test's `DBOS` with `migrate: false`.
- **Simulating crash recovery in tests**: A single `DBOS` instance supports `launch()` -> `shutdown()` -> `launch()` cycles, or you can construct a second `DBOS` instance with the same `(app_name, app_version, executor_id, database_url)` to verify that `launch()` recovers `PENDING` rows left behind when the first instance shut down immediately (`dbos.shutdown().await`).

Reference: [dbos::DBOS::shutdown](https://docs.rs/dbos/latest/dbos/struct.DBOS.html#method.shutdown), [dbos::sysdb::migrations::runner::run](https://docs.rs/dbos/latest/dbos/sysdb/migrations/runner/fn.run.html)
