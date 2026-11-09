---
title: Use Proper Test Setup for DBOS
impact: LOW-MEDIUM
impactDescription: Tests that share IDs or instances interfere with each other
tags: testing, tokio-test, integration
---

## Use Proper Test Setup for DBOS

Outside a workflow, `dbos::step` and `dbos::sleep` run as plain calls, so functions built from steps are unit-testable without a database. To test durable behavior, launch DBOS against a test database inside each test, with fresh workflow IDs (an ID that exists returns the existing workflow), and shut it down at the end. Each `#[tokio::test]` has its own runtime, so don't share a `DBOS` across tests through a `static`. Tests run in parallel, and instances that share a database, application version and executor ID (`"local"` by default) interfere: launching re-enqueues the `PENDING` workflows recorded under that executor ID and version, including another test's running ones. Fresh workflow IDs don't prevent this. Give each test its own database or `Config::schema`, or run tests that share one with `--test-threads=1`.

**Incorrect (calling set_event outside a workflow in a unit test):**

```rust
#[tokio::test]
async fn publishes_status() {
    // WRONG: outside a workflow, set_event returns Error::NotInWorkflow.
    dbos::set_event::<_, dbos::EngineOnly>("status", &"done").await.unwrap();
}
```

**Correct (a unit test and an integration test):**

```rust
#[tokio::test]
async fn quote_applies_the_discount() {
    // Outside a workflow, each step runs as a plain function call and records nothing.
    assert_eq!(quote("widget".to_owned()).await.unwrap(), 90);
}
```

```rust
use dbos::{Config, DBOS, StartOptions};

async fn launch_for_test() -> dbos::Result<(DBOS, dbos::WorkflowRef<String, u32>)> {
    // A database for tests alone: DBOS creates it if it doesn't exist.
    let mut config = Config::new(
        "pricing-tests",
        std::env::var("TEST_DATABASE_URL")
            .unwrap_or_else(|_| "postgres://postgres:dbos@localhost:5432/pricing_test".to_owned()),
    );
    config.app_version = Some("test".to_owned());
    let dbos = DBOS::new(config);
    let quote = dbos.register_workflow("quote", quote)?;
    dbos.launch().await?;
    Ok((dbos, quote))
}

#[tokio::test]
async fn quote_runs_as_a_workflow() -> dbos::Result<()> {
    let (dbos, quote) = launch_for_test().await?;

    let workflow_id = format!("quote-test-{}", uuid::Uuid::new_v4());
    let options = StartOptions {
        workflow_id: Some(&workflow_id),
        ..StartOptions::default()
    };
    let handle = quote.start_with("gadget".to_owned(), options).await?;
    assert_eq!(handle.result().await?, 225);

    // The workflow recorded both steps.
    let steps = dbos.list_workflow_steps(&workflow_id).await?;
    let names: Vec<_> = steps.iter().map(|step| step.step_name.as_str()).collect();
    assert_eq!(names, ["fetch_price", "apply_discount"]);

    dbos.shutdown().await;
    Ok(())
}
```
