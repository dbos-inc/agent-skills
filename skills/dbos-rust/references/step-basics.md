---
title: Wrap External I/O and Side Effects in Steps
impact: HIGH
impactDescription: Checkpoints external results in PostgreSQL so recovered workflows skip completed side effects
tags: step, checkpoint, external, io, FnMut, cancellation_token
---

## Wrap External I/O and Side Effects in Steps

Execute every external API call, database query, filesystem operation, clock read, or non-deterministic calculation inside `dbos::step` (or `dbos::step_with`). When a workflow recovers after a crash, DBOS returns the checkpointed result for every completed step from PostgreSQL instead of running its closure again.

**Incorrect (performing external HTTP call directly in workflow and consuming captured String with FnOnce):**

```rust
async fn charge_customer(customer_id: String, cents: u32) -> dbos::Result<String> {
    // External call directly in workflow body: runs again on every crash recovery!
    let token = fetch_payment_token(&customer_id).await;

    // Fails to compile: dbos::step requires FnMut() -> Fut because retries call the closure
    // more than once, so moving `customer_id` out of the closure makes it FnOnce!
    dbos::step("charge_card", move || async move {
        submit_charge(customer_id, &token, cents).await
    })
    .await
}
```

**Correct (all I/O in steps, borrowing or cloning per attempt to satisfy FnMut):**

```rust
async fn fetch_payment_token(customer_id: &str) -> dbos::Result<String> {
    Ok(format!("tok_{customer_id}"))
}

async fn submit_charge(customer_id: &str, token: &str, cents: u32) -> dbos::Result<String> {
    Ok(format!("receipt_{customer_id}_{token}_{cents}"))
}

async fn charge_customer(customer_id: String, cents: u32) -> dbos::Result<String> {
    let token = dbos::step("fetch_payment_token", || fetch_payment_token(&customer_id)).await?;

    let receipt = dbos::step("charge_card", || {
        submit_charge(&customer_id, &token, cents)
    })
    .await?;

    Ok(receipt)
}
```

### Step Mechanics and Rules

- **`FnMut` closure bound**: `dbos::step` and `dbos::step_with` require `F: FnMut() -> Fut + Send` because retry attempts invoke the closure repeatedly. Borrow outer variables (`&customer_id`) or clone inside the outer `||` closure before moving into an `async move` block (`|| { let id = id.clone(); async move { ... } }`).
- **Serializable outputs and errors**: Step return values must implement `Serialize + DeserializeOwned + Send`, and custom error types must implement `DurableError` (`std::error::Error + Serialize + DeserializeOwned + 'static`).
- **Leaf execution rule**: Calling `dbos::step` inside an already-running step body executes it as a plain async call without allocating a new `step_id` or writing a second checkpoint. Calling `dbos::step` outside any workflow also runs plainly, making step helpers directly unit-testable without launching `DBOS`.
- **Step names are verified on replay**: Replaying a workflow checks that the step name at each `step_id` matches the recorded name, returning `sysdb::Error::UnexpectedStep` if code changed without patching.
- **Cooperative cancellation for blocking work**: Inside a step that offloads CPU or synchronous I/O to `tokio::task::spawn_blocking`, clone `dbos::cancellation_token()` before spawning so the blocking loop can stop promptly when a step timeout, `select_step!` race loss, or `preemptible: true` cancellation abandons the attempt.

Reference: [dbos::step](https://docs.rs/dbos/latest/dbos/fn.step.html), [dbos::cancellation_token](https://docs.rs/dbos/latest/dbos/fn.cancellation_token.html)
