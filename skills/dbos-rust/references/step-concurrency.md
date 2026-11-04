---
title: Run Steps Concurrently with join!
impact: HIGH
impactDescription: Sequential independent steps waste time; unrecorded concurrency breaks recovery
tags: step, concurrency, join, parallel
---

## Run Steps Concurrently with join!

Build several steps, then join them with `tokio::join!` or `futures::future::try_join_all`. Each step takes its ID where it is built, so the IDs are the same on every recovery, and the join returns results in build order. To race steps, use `select_step!` (see `workflow-concurrency.md`).

**Incorrect (awaiting independent steps one at a time):**

```rust
async fn two_fetches(_: ()) -> dbos::Result<(String, String)> {
    // Slow: the second fetch waits for the first.
    let a = step("fetch_a", || fetch("https://a.example".to_owned())).await?;
    let b = step("fetch_b", || fetch("https://b.example".to_owned())).await?;
    Ok((a, b))
}
```

**Correct (build, then join):**

```rust
async fn fan_out(urls: Vec<String>) -> dbos::Result<Vec<String>> {
    // Each step takes its ID here, in the order of `urls`.
    let fetches: Vec<_> = urls
        .iter()
        .map(|url| step("fetch", move || fetch(url.clone())))
        .collect();
    // The steps run concurrently, and the results come back in the same order.
    futures::future::try_join_all(fetches).await
}

async fn two_at_once(_: ()) -> dbos::Result<(String, String)> {
    let (a, b) = tokio::join!(
        step("fetch_a", || fetch("https://a.example".to_owned())),
        step("fetch_b", || fetch("https://b.example".to_owned())),
    );
    Ok((a?, b?))
}
```

```rust
async fn build_order(_: ()) -> dbos::Result<()> {
    let first = step("first", || async { Ok(1) }); // step ID 0
    let second = step("second", || async { Ok(2) }); // step ID 1
    // Awaiting them in the other order doesn't change their IDs.
    second.await?;
    first.await?;
    Ok(())
}
```
