---
title: Share a System Database Between Applications
impact: LOW
impactDescription: Applications sharing a database without names run each other's work
tags: advanced, shared-database, app_name, multi-app
---

## Share a System Database Between Applications

Several applications (in any language) can share one system database, isolated by `app_name`. Each queue and schedule belongs to the application that registers it. To run another application's workflow, enqueue it by name with `app_name` set to the application that implements it.

**Incorrect (a nameless client writing to a shared database):**

```rust
// WRONG: with several applications on one database, a nameless client
// writes unclaimed rows that any application may run.
let client = Client::connect(ClientConfig::from_env()).await?;
client.enqueue::<_, String, dbos::EngineOnly>("process_order", "orders", "order-123").await?;
```

**Correct (naming the target application):**

```rust
async fn call_order_service(dbos: &DBOS) -> dbos::Result<()> {
    let options = EnqueueOptions {
        // The name of the application that implements process_order
        app_name: Some("order-service"),
        ..EnqueueOptions::new("orders")
    };
    let handle: WorkflowHandle<String> = dbos
        .enqueue_with("process_order", "order-123", options)
        .await?;
    let result = handle.result().await?;
    Ok(())
}
```
