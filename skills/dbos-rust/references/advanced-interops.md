---
title: Use Portable Serialization for Cross-Language DBOS Interop
impact: LOW
impactDescription: Exchanges workflows, events, messages, and streams seamlessly with Python, TypeScript, Go, Java, and SQL clients
tags: advanced, interop, SerializationFormat, Portable, PortableWorkflowArgs, cross-language
---

## Use Portable Serialization for Cross-Language DBOS Interop

DBOS SDKs across Rust, Python, TypeScript, Go, Java, and SQL (`dbos.enqueue_workflow`) share the `"portable_json"` wire format (`SerializationFormat::Portable`). Register cross-language Rust workflows with `WorkflowOptions { serialization: SerializationFormat::Portable }`, and use `SerializationFormat::Portable` or `PortableWorkflowArgs` when a Rust `DBOS` or `Client` invokes or signals workflows written in another language.

**Incorrect (using native rust_serde for cross-language calls or passing [T; N] expecting one list arg):**

```rust
use dbos::{Client, Enqueue, EnqueueOptions, SerializationFormat};

async fn broken_cross_sdk_enqueue(client: &Client) -> dbos::Result<()> {
    // Trap 1: Default `SerializationFormat::Default` writes `"rust_serde"` instead of `"portable_json"`,
    // which Python/TypeScript/Go/Java workers do not treat as portable args!
    let _h1 = client
        .enqueue::<_, String, dbos::EngineOnly>("python_worker", "ml-queue", "hello".to_owned())
        .await?;

    // Trap 2: In `"portable_json"`, a Rust tuple `(A, B)` or fixed array `[T; N]` (`N >= 2`) spreads
    // into N separate `positionalArgs`! Passing `[10, 20, 30]` calls `fn(10, 20, 30)` instead of `fn(items)`!
    let _h2 = client
        .enqueue_with::<_, i64, dbos::EngineOnly>(
            "sum_list",
            [10_i64, 20, 30],
            EnqueueOptions::on(Enqueue {
                serialization: SerializationFormat::Portable,
                ..Enqueue::new("ml-queue")
            }),
        )
        .await?;
    Ok(())
}
```

**Correct (registering portable Rust workflows and dispatching typed or named portable args):**

```rust
use dbos::{
    Client, DBOS, EngineOnly, Enqueue, EnqueueOptions, Error, PortableWorkflowArgs,
    SerializationFormat, WorkflowOptions, WorkflowRef,
};
use serde::{Deserialize, Serialize};
use serde_json::json;

#[derive(Debug, Serialize, Deserialize)]
struct OrderRequest {
    sku: String,
    qty: u32,
}

// 1. Register a Rust workflow callable from Python / TypeScript / Go / Java / SQL:
//    - Single positional object OR Python `namedArgs` (`sku="...", qty=2`) decodes into `OrderRequest`!
//    - Events, messages, and stream chunks emitted inside it inherit `"portable_json"` automatically.
fn register_portable_rust_workflow(
    dbos: &DBOS,
) -> dbos::Result<WorkflowRef<OrderRequest, String>> {
    dbos.register_workflow_with(
        "fulfill_order",
        WorkflowOptions {
            serialization: SerializationFormat::Portable,
        },
        |req: OrderRequest| async move {
            dbos::set_event("order_sku", &req.sku).await?;
            Ok(format!("fulfilled {} x {}", req.qty, req.sku))
        },
    )
}

// 2. Enqueue a multi-arg or keyword-arg Python/TypeScript workflow from Rust and handle foreign errors:
async fn call_python_workflow(client: &Client) -> dbos::Result<String> {
    // Use `Vec<T>` when passing a single list argument; use `(A, B)` to pass 2 positional arguments;
    // or use `PortableWorkflowArgs` when mixing positionalArgs + Python `**kwargs` (`namedArgs`)!
    let mut args = PortableWorkflowArgs {
        positional_args: vec![json!(["item-a", "item-b"])],
        ..PortableWorkflowArgs::default()
    };
    args.named_args
        .insert("dry_run".to_owned(), json!(false));

    let handle = client
        .enqueue_portable::<String, EngineOnly>(
            "run_ml_pipeline",
            args,
            EnqueueOptions::on(Enqueue::new("ml-queue")),
        )
        .await?;

    match handle.result().await {
        Ok(summary) => Ok(summary),
        // Exceptions raised by Python/TS/Go/Java workers arrive as `Error::Portable`
        Err(Error::Portable {
            name,
            message,
            code,
            ..
        }) => Ok(format!("remote error {name} ({code:?}): {message}")),
        Err(other) => Err(other),
    }
}
```

### Portable Serialization Rules

- **Argument mapping (`encode_input` / `decode_input`)**:
  - `()` <-> `{"positionalArgs":[],"namedArgs":{}}`
  - `(A, B, ...)` (`arity >= 2`) <-> `"positionalArgs":[a, b, ...]`
  - Struct, `Vec<T>`, scalar, or `(x,)` <-> `"positionalArgs":[value]` (on decode, `"namedArgs":{"k":v}` with empty `positionalArgs` also decodes directly into a Rust struct!).
  - `PortableWorkflowArgs` <-> raw envelope (required when **both** `positionalArgs` and `namedArgs` are non-empty).
- **Strict JSON constraints**: `"portable_json"` rejects non-finite floats (`NaN`, `Infinity`, `-Infinity`) and non-string map keys with `Error::Serialization`.

Reference: [dbos::SerializationFormat](https://docs.rs/dbos/latest/dbos/enum.SerializationFormat.html), [dbos::PortableWorkflowArgs](https://docs.rs/dbos/latest/dbos/struct.PortableWorkflowArgs.html), [dbos::Client::enqueue_portable](https://docs.rs/dbos/latest/dbos/struct.Client.html#method.enqueue_portable), [Portable Workflows](https://docs.dbos.dev/explanations/portable-workflows)
