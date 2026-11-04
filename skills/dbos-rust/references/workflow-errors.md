---
title: Give Workflows a Recordable Error Type
impact: HIGH
impactDescription: A workflow whose error type cannot be serialized does not register
tags: workflow, errors, serde, thiserror
---

## Give Workflows a Recordable Error Type

A workflow returns `dbos::Result<T, E>`, which is `Result<T, dbos::Error<E>>`. `E` is the application's error type and must implement `std::error::Error`, `Serialize` and `Deserialize` (the `DurableError` bound), because DBOS records the error and replays it. `dbos::Result<T>` (no `E`) can carry only DBOS's own errors. `?` converts `E` into `dbos::Error::Application(E)`.

**Incorrect (an error type DBOS cannot record):**

```rust
// WRONG: register_workflow requires the error type to be Serialize + Deserialize +
// std::error::Error, so it can be recorded and replayed. This doesn't register.
async fn checkout(item: String) -> Result<String, Box<dyn std::error::Error + Send + Sync>> {
    Err(format!("item {item} is out of stock").into())
}
```

**Correct (a serde-derived thiserror enum):**

```rust
#[derive(Debug, thiserror::Error, serde::Serialize, serde::Deserialize)]
enum CheckoutError {
    #[error("item {0} is out of stock")]
    OutOfStock(String),
}

async fn checkout(item: String) -> dbos::Result<String, CheckoutError> {
    // A step over a dbos::Result<T> function joins the workflow's error type with lift().
    let reserved = step("reserve", || reserve(&item)).lift().await?;
    if !reserved {
        // `?` converts CheckoutError into dbos::Error::Application.
        Err(CheckoutError::OutOfStock(item.clone()))?;
    }
    Ok(format!("reserved {item}"))
}
```

DBOS's own calls (`dbos::sleep`, `send`, `recv`, `set_event`, ...) take the workflow's error type, so `?` works on them directly. A step takes its error type from its function: carry a `dbos::Result<T>` step into another error type with `.lift()`, or convert with `.map_error()`. Return `Error::WorkflowCancelled` when a DBOS call reports it; don't swallow it. `dbos::Error` is `#[non_exhaustive]`; match with a wildcard arm.
