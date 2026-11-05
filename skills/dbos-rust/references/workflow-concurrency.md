---
title: Race and Join Durable Calls Only with DBOS Primitives
impact: CRITICAL
impactDescription: Races, timeouts, completion-order streams and spawns over DBOS calls silently replay differently
tags: workflow, concurrency, select, join, tokio, futures, smol, determinism
---

## Race and Join Durable Calls Only with DBOS Primitives

Every DBOS call takes its step ID when it is built, not when it is awaited, so joining DBOS calls is deterministic when each join element is one call, not an `async` block containing several, whose calls take IDs in the order the blocks run. The exceptions are `dbos::patch` and `dbos::deprecate_patch`, which claim their positions when awaited, and a debouncer's `debounce`, which can claim more when awaited: await those on their own, never inside a join or between building and awaiting another call. Racing them is not: nothing records which branch won. In a workflow body, do NOT use `tokio::select!`, `futures::select!`/`select_biased!`, `futures::future::select`/`try_select`/`select_all`/`select_ok`, `futures::stream::select`/`select_all`, `smol::future::race`/`or`, `smol::stream::race`/`or`, `tokio::time::timeout`/`timeout_at`, or an `async_io::Timer` race around DBOS calls. Do NOT consume DBOS calls in completion order with `FuturesUnordered`, `buffer_unordered` or `for_each_concurrent`. Do NOT spawn them with `tokio::spawn`, `spawn_local`, `LocalSet`, `JoinSet`, `smol::spawn`, `async_global_executor::spawn` or `std::thread::spawn`: a spawned task doesn't inherit the workflow. Built inside it, a step, `dbos::sleep`, a handle's `result()` or a `DBOS` method silently runs unrecorded; `start` and `run` silently start a new top-level workflow on each recovery that repeats the spawn; and `send`/`recv`/`set_event`/`get_event` fail with `Error::NotInWorkflow`. Built in the workflow and awaited in it, any of them fails with `Error::StepBuiltElsewhere`. Do NOT use `tokio::time::sleep`, `sleep_until` or `interval` as a delay in a workflow body.

**Incorrect (unrecorded race, completion order, and a spawn):**

```rust
async fn pay_or_expire(_: ()) -> dbos::Result<String> {
    // WRONG: tokio::select! doesn't record which branch won,
    // so a recovery can take the other branch.
    tokio::select! {
        paid = step("wait_for_payment", wait_for_payment) => paid,
        _ = tokio::time::sleep(Duration::from_secs(3600)) => Ok("expired".to_owned()),
    }
}

async fn fan_out(urls: Vec<String>) -> dbos::Result<()> {
    use futures::StreamExt;
    // WRONG: results arrive in completion order, which a recovery doesn't reproduce,
    // so the per-result steps are handed each other's recorded outputs.
    let mut pending: futures::stream::FuturesUnordered<_> = urls
        .iter()
        .map(|url| step("fetch", move || fetch(url.clone())))
        .collect();
    while let Some(page) = pending.next().await {
        let page = page?;
        step("store", || async { Ok(page.len()) }).await?;
    }
    // WRONG: a step built here and awaited in a spawned task fails with
    // Error::StepBuiltElsewhere; one built inside the task runs unrecorded.
    tokio::spawn(step("background", fetch_data));
    Ok(())
}
```

**Correct (select_step! and an ordered join):**

```rust
async fn pay_or_expire(_: ()) -> dbos::Result<String> {
    // select_step! records the winner, so a recovery takes the same arm.
    dbos::select_step! {
        paid = step("wait_for_payment", wait_for_payment) => paid?,
        expired = dbos::sleep(Duration::from_secs(3600)) => {
            expired?;
            "expired".to_owned()
        }
    }
}

async fn fan_out(urls: Vec<String>) -> dbos::Result<()> {
    // Steps take their IDs where they're built, in order, and
    // try_join_all returns the results in that same order.
    let fetches: Vec<_> = urls
        .iter()
        .map(|url| step("fetch", move || fetch(url.clone())))
        .collect();
    let pages = futures::future::try_join_all(fetches).await?;
    for page in pages {
        step("store", || async { Ok(page.len()) }).await?;
    }
    Ok(())
}
```

Supported in a workflow body: `tokio::join!`, `futures::join!`, `futures::future::join_all`, `futures_lite::future::zip`, the first-failure forms `tokio::try_join!`, `futures::try_join!`, `futures::future::try_join_all` and `futures_lite::future::try_zip` only when the error goes straight out of the workflow (nothing records which error came back, so never branch on it), `dbos::select_step!`, `dbos::select_workflow` (recorded) and `dbos::join_workflows` (not recorded; a recovery waits again), their macro forms, and `dbos::sleep`.

`select_step!` branches are each exactly one DBOS call (a step, `dbos::sleep`, `recv`, `get_event`, a handle's `result()`). It refuses `if` guards, `else`, `biased;` and fewer than two branches at compile time, and polls branches in source order. A losing step records nothing; a losing `result()` leaves the child running. `start` and `run` are refused as branches by type: start the child before the race and race its `result()`.

There is no durable completion-order stream yet: run such fan-outs as child workflows on a queue, or sequentially. Inside a step, plain races, timeouts and spawns are fine, but a DBOS call spawned from a step fails.
