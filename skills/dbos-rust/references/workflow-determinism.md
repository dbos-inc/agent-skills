---
title: Keep Workflows Deterministic
impact: CRITICAL
impactDescription: Non-deterministic branches or step ordering cause replay mismatches and UnexpectedStep errors on recovery
tags: workflow, determinism, replay, recovery, step
---

## Keep Workflows Deterministic

When DBOS recovers an interrupted workflow, it re-executes the workflow function from the beginning and replays recorded step outputs by their sequential 0-based `step_id` (`0, 1, 2, ...`). Given the same workflow input and recorded step results, the workflow body must invoke the exact same sequence of durable operations in the exact same order on every run.

**Incorrect (clock reads, random IDs, HashMap iteration, and tokio::select! in workflow body):**

```rust
use std::collections::HashMap;
use std::time::{Duration, SystemTime};
use uuid::Uuid;

async fn process_batch(items: HashMap<String, u32>) -> dbos::Result<String> {
    // Non-deterministic: generates a different UUID on replay!
    let batch_id = Uuid::new_v4().to_string();

    // Non-deterministic: SystemTime changes on replay, flipping the branch!
    if SystemTime::now()
        .duration_since(SystemTime::UNIX_EPOCH)
        .unwrap_or_default()
        .as_secs()
        % 2
        == 0
    {
        dbos::step("even_branch", || async { Ok(()) }).await?;
    }

    // Non-deterministic: HashMap randomized iteration order scrambles step IDs!
    for (key, qty) in &items {
        dbos::step("ship_item", || async move { Ok((*key).clone()) }).await?;
        let _ = qty;
    }

    // Non-deterministic: tokio::select! does not record which branch won!
    tokio::select! {
        res = dbos::step("primary", || async { Ok("ok".to_owned()) }) => res,
        () = tokio::time::sleep(Duration::from_secs(5)) => Ok(batch_id),
    }
}
```

**Correct (isolate non-determinism in steps, sort keys, and use durable primitives):**

```rust
use std::collections::BTreeMap;
use std::time::{Duration, SystemTime, UNIX_EPOCH};
use uuid::Uuid;

async fn process_batch(items: BTreeMap<String, u32>) -> dbos::Result<String> {
    // Recorded at step 0: replay returns the original UUID and timestamp
    let (batch_id, started_secs) = dbos::step("init_metadata", || async {
        let id = Uuid::new_v4().to_string();
        let secs = SystemTime::now()
            .duration_since(UNIX_EPOCH)
            .unwrap_or_default()
            .as_secs();
        Ok((id, secs))
    })
    .await?;

    // Deterministic: branches on the checkpointed value from step 0
    if started_secs % 2 == 0 {
        dbos::step("even_branch", || async { Ok(()) }).await?;
    }

    // Deterministic: BTreeMap iterates in sorted key order on every run
    for (key, qty) in &items {
        let item_key = key.clone();
        let item_qty = *qty;
        dbos::step("ship_item", || {
            let item_key = item_key.clone();
            async move { Ok((item_key, item_qty)) }
        })
        .await?;
    }

    // Deterministic: select_step! checkpoints the winning arm index for replay
    dbos::select_step! {
        res = dbos::step("primary", || async { Ok("ok".to_owned()) }) => res?,
        slept = dbos::sleep(Duration::from_secs(5)) => {
            slept?;
            batch_id
        }
    }
}
```

### Operations That Must Live Inside Steps or Durable Primitives

- **Wall-clock and monotonic time**: Wrap `SystemTime::now()`, `Instant::now()`, or `jiff::Timestamp::now()` in `dbos::step`, and use `dbos::sleep` instead of `tokio::time::sleep` (see [pattern-sleep.md](pattern-sleep.md)).
- **Randomness and generated IDs**: Wrap `rand`, `Uuid::new_v4()`, or cryptographic nonces inside `dbos::step` (or derive deterministic keys from `dbos::workflow_id()`).
- **Network, filesystem, environment, and database I/O**: All HTTP requests, file reads/writes, runtime env lookups that can drift across restarts, and SQL queries belong inside `dbos::step`.
- **Collection order**: Use `Vec` or `BTreeMap` / `BTreeSet` (or sort keys explicitly) before looping over durable calls; `HashMap` and `HashSet` use randomized `SipHash` seeds per process.
- **Racing concurrent operations**: Never race steps with `tokio::select!`; use `dbos::select_step!` or `dbos::select_workflow!` so the winning branch is checkpointed (see [step-concurrency.md](step-concurrency.md)).

When a replaying workflow attempts to execute a step whose name disagrees with the step recorded at that `step_id`, DBOS refuses with `sysdb::Error::UnexpectedStep`. Keep step names stable, and use `dbos::patch` when changing the step sequence of deployed workflows (see [advanced-patching.md](advanced-patching.md)).

Reference: [dbos crate documentation](https://docs.rs/dbos/latest/dbos/), [How Workflow Recovery Works](https://docs.dbos.dev/architecture#how-workflow-recovery-works)
