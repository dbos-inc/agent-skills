---
title: Register Class-Scoped and Multi-Instance Workflows with QualifiedWorkflowName
impact: LOW-MEDIUM
impactDescription: Distinguishes multiple configured instances of the same workflow logic and matches class-bound workflows across SDKs
tags: workflow, instances, QualifiedWorkflowName, register_workflow, class_name, config_name
---

## Register Class-Scoped and Multi-Instance Workflows with QualifiedWorkflowName

In DBOS Rust, workflow registration takes `impl Into<QualifiedWorkflowName>`, which carries three components stored on every workflow row: `(name, class_name, config_name)`. While a plain `&str` registers a free workflow (`class_name = None`, `config_name = None`), use `QualifiedWorkflowName::method(name, class_name)` for class-grouped workflows and `QualifiedWorkflowName::instance(name, class_name, config_name)` when registering multiple pre-configured instances (for example, per-region or per-provider endpoints) of the same workflow function before `dbos.launch()`.

**Incorrect (concatenating ad-hoc strings or setting config_name without a class_name):**

```rust
use dbos::{Client, DBOS, QualifiedWorkflowName};

async fn mismatched_registration(dbos: &DBOS, client: &Client) -> dbos::Result<()> {
    // Registers `name = "charge_stripe_eu"`, `class_name = None`, `config_name = None`
    let _eu = dbos.register_workflow("charge_stripe_eu", |cents: u64| async move {
        Ok::<_, dbos::Error>(cents)
    })?;

    dbos.launch().await?;

    // Mismatch! Enqueues `name = "charge"`, `class_name = Some("PaymentGateway")`,
    // `config_name = Some("stripe-eu")`, which no registered workflow matches!
    let _handle = client
        .enqueue::<_, u64, dbos::EngineOnly>(
            QualifiedWorkflowName::instance("charge", "PaymentGateway", "stripe-eu"),
            "payments",
            5000_u64,
        )
        .await?;
    Ok(())
}
```

**Correct (registering per-instance closures before launch with QualifiedWorkflowName::instance):**

```rust
use std::sync::Arc;
use dbos::{Client, DBOS, QualifiedWorkflowName, WorkflowRef};

struct GatewayClient {
    region: String,
    api_base: String,
}

impl GatewayClient {
    async fn charge(&self, cents: u64) -> dbos::Result<String> {
        let endpoint = format!("{}/charges", self.api_base);
        let region = self.region.clone();
        dbos::step("post_charge", || {
            let endpoint = endpoint.clone();
            let region = region.clone();
            async move { Ok(format!("{region}:{endpoint}:{cents}")) }
        })
        .await
    }
}

fn register_gateway_instance(
    dbos: &DBOS,
    instance_name: &str,
    gateway: Arc<GatewayClient>,
) -> dbos::Result<WorkflowRef<u64, String>> {
    dbos.register_workflow(
        QualifiedWorkflowName::instance("charge", "GatewayClient", instance_name),
        move |cents: u64| {
            let gateway = Arc::clone(&gateway);
            async move { gateway.charge(cents).await }
        },
    )
}

async fn setup_and_enqueue(dbos: &DBOS, client: &Client) -> dbos::Result<()> {
    // Register every configured instance BEFORE `dbos.launch()`
    let _us = register_gateway_instance(
        dbos,
        "us-east",
        Arc::new(GatewayClient {
            region: "us-east".to_owned(),
            api_base: "https://us.pay.example.com".to_owned(),
        }),
    )?;
    let _eu = register_gateway_instance(
        dbos,
        "eu-west",
        Arc::new(GatewayClient {
            region: "eu-west".to_owned(),
            api_base: "https://eu.pay.example.com".to_owned(),
        }),
    )?;

    dbos.launch().await?;

    // External `Client` (or Python/TypeScript/Java SDK) targets the exact instance
    let handle = client
        .enqueue::<_, String, dbos::EngineOnly>(
            QualifiedWorkflowName::instance("charge", "GatewayClient", "eu-west"),
            "payments",
            2500_u64,
        )
        .await?;
    let _receipt = handle.result().await?;
    Ok(())
}
```

### Rules and Constraints for `QualifiedWorkflowName`

- **Three constructors**:
  - `QualifiedWorkflowName::new("name")` (or `"name"` directly via `From<&str>`): free workflow.
  - `QualifiedWorkflowName::method("name", "ClassName")`: class-scoped workflow with no instance config.
  - `QualifiedWorkflowName::instance("name", "ClassName", "config-key")`: configured instance of a class.
- **Pre-launch registration required**: Because `dbos.launch()` snapshots the workflow registry for recovery and queue workers, every configured instance must be registered before `launch()`.
- **Schedules cannot target configured instances**: `workflow_schedules` has no `config_name` column, so `ScheduleSpec::new` and `ScheduleSpec::for_workflow` reject `QualifiedWorkflowName::instance(...)` with `Error::InvalidArgument` (`new` and `method` both work).

Reference: [dbos::QualifiedWorkflowName](https://docs.rs/dbos/latest/dbos/sysdb/types/struct.QualifiedWorkflowName.html), [dbos::DBOS::register_workflow](https://docs.rs/dbos/latest/dbos/struct.DBOS.html#method.register_workflow)
