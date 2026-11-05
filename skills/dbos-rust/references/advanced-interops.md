---
title: Cross-Language Workflows with Portable JSON
impact: LOW
impactDescription: Workflows started or read by other languages cannot decode Rust's default encoding
tags: advanced, portable, interop, cross-language
---

## Cross-Language Workflows with Portable JSON

Register a workflow that other languages start or read with `WorkflowOptions { serialization: SerializationFormat::Portable, .. }`. Single calls (enqueue, send, set_event, write_stream) take `serialization: SerializationFormat::Portable` in their options. A tuple of two or more is sent as that many positional arguments; `()` as none. Return `dbos::Error::Portable { name, message, code, data }` to choose the error envelope.

**Incorrect (the default encoding for a cross-language workflow):**

```rust
// WRONG: rust_serde rows aren't readable by Python, TypeScript, Go or Java.
let greet = dbos.register_workflow("greet", greet)?;
```

**Correct (a portable registration):**

```rust
fn register_portable(dbos: &DBOS) -> dbos::Result<()> {
    // Other languages can start this workflow and read its result.
    let greet = dbos.register_workflow_with(
        "greet",
        WorkflowOptions {
            serialization: SerializationFormat::Portable,
            ..WorkflowOptions::default()
        },
        |name: String| async move { Ok::<_, dbos::Error>(format!("hello, {name}")) },
    )?;
    Ok(())
}
```

```rust
async fn enqueue_portable() -> dbos::Result<()> {
    let client = Client::connect(ClientConfig::from_env()).await?;
    let options = EnqueueOptions {
        // The name of the application that implements process_order
        app_name: Some("order-service"),
        serialization: SerializationFormat::Portable,
        ..EnqueueOptions::new("orders")
    };
    let handle: WorkflowHandle<String> = client
        .enqueue_with("process_order", "order-123", options)
        .await?;
    Ok(())
}
```
