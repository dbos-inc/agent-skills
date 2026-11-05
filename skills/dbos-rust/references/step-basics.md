---
title: Use Steps for External Operations
impact: HIGH
impactDescription: Unrecorded side effects run again on every recovery
tags: step, external, api, at-least-once
---

## Use Steps for External Operations

Run anything that talks to the outside world or is non-deterministic as a step: `dbos::step(name, || f(args))`. The step's output must be `Serialize + Deserialize`. The name is checked on recovery, so keep it stable. The closure is `FnMut` because retries call it again: capture by reference or clone inside it.

**Incorrect (an API call in the workflow body):**

```rust
async fn workflow_function(url: String) -> dbos::Result<String> {
    // WRONG: not a step, so it runs again on every recovery.
    let body = reqwest::get(&url).await.unwrap().text().await.unwrap();
    Ok(body)
}
```

**Correct (the call as a step):**

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

Steps are at-least-once: a crash after the side effect but before the checkpoint runs the step again. DBOS for Rust has no transactional step, so make database writes in steps idempotent (unique keys, `ON CONFLICT DO NOTHING`). A step called from another step runs as a plain call inside it. A step's body may spawn tasks, race futures and use timeouts freely.
