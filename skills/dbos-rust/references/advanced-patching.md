---
title: Evolve In-Flight Workflows Safely with patch and deprecate_patch
impact: LOW
impactDescription: Allows adding, removing, or reordering steps under the same app_version without breaking in-flight executions
tags: advanced, patch, deprecate_patch, upgrade, compatibility, replay
---

## Evolve In-Flight Workflows Safely with patch and deprecate_patch

Because recovery replays recorded steps by ordinal position (`0, 1, 2, ...`), inserting, deleting, or reordering durable calls inside an existing workflow breaks in-flight executions running under the same `Config::app_version`. When keeping `app_version` unchanged across a deploy, guard workflow structure changes with `dbos::patch("name").await?`, then transition through `dbos::deprecate_patch("name").await?` once older runs have finished.

**Incorrect (inserting a new step directly in place while old runs are still active):**

```rust
// Previously deployed v1:
//   step 0: charge_card
//   step 1: ship_order

// Deployed under the same app_version while v1 workflows are still in flight:
async fn fulfill_order(order_id: String) -> dbos::Result<()> {
    // Replaying a v1 workflow that already completed `charge_card` at step 0
    // hits `verify_address` at step 0 instead and fails with UnexpectedStep!
    dbos::step("verify_address", || verify_address(&order_id)).await?;
    dbos::step("charge_card", || charge_card(&order_id)).await?;
    dbos::step("ship_order", || ship_order(&order_id)).await
}
```

**Correct (three-stage rollout using patch, deprecate_patch, and final removal):**

```rust
async fn verify_address(_order_id: &str) -> dbos::Result<()> {
    Ok(())
}

async fn charge_card(_order_id: &str) -> dbos::Result<()> {
    Ok(())
}

async fn ship_order(_order_id: &str) -> dbos::Result<()> {
    Ok(())
}

// Stage 1: Deploy with `dbos::patch` while pre-patch runs may still recover.
// - New runs (or old runs that had not reached step 0 yet) write marker `DBOS.patch-verify-v1`
//   and return `Ok(true)`.
// - Old runs that already recorded `charge_card` at step 0 return `Ok(false)` WITHOUT advancing
//   the step counter, so the `else` branch replays `charge_card` at step 0 cleanly.
async fn fulfill_order_stage1(order_id: String) -> dbos::Result<()> {
    if dbos::patch("verify-v1").await? {
        dbos::step("verify_address", || verify_address(&order_id)).await?;
    }
    dbos::step("charge_card", || charge_card(&order_id)).await?;
    dbos::step("ship_order", || ship_order(&order_id)).await
}

// Stage 2: Once zero pre-patch workflows remain active, replace `patch` with `deprecate_patch`.
// Runs carrying the `DBOS.patch-verify-v1` marker consume it; brand-new runs write no marker.
async fn fulfill_order_stage2(order_id: String) -> dbos::Result<()> {
    dbos::deprecate_patch("verify-v1").await?;
    dbos::step("verify_address", || verify_address(&order_id)).await?;
    dbos::step("charge_card", || charge_card(&order_id)).await?;
    dbos::step("ship_order", || ship_order(&order_id)).await
}

// Stage 3: Once all Stage 1 workflows carrying the marker have completed, delete `deprecate_patch`.
async fn fulfill_order_stage3(order_id: String) -> dbos::Result<()> {
    dbos::step("verify_address", || verify_address(&order_id)).await?;
    dbos::step("charge_card", || charge_card(&order_id)).await?;
    dbos::step("ship_order", || ship_order(&order_id)).await
}
```

### Patching Rules in DBOS Rust

- **No `enable_patching` configuration flag**: `dbos::patch` and `dbos::deprecate_patch` work out of the box with no `Config` toggle.
- **Keep `Config::app_version` constant when patching**: Executors only recover and dequeue workflows matching their own `app_version`. If you bump `app_version` on every deploy and drain old executors (blue-green deployment), you do not need patches at all (see [advanced-versioning.md](advanced-versioning.md)).
- **Await patches immediately in sequential workflow code**: `patch` and `deprecate_patch` peek the current `step_id` when constructed and verify it has not moved when polled. Constructing another durable call before `.await`ing a patch (or placing a patch inside `tokio::join!`) fails with `Error::PatchNondeterminism`. Neither function returns `PendingStep`, so patches cannot be raced inside `select_step!`.
- **One-way compatibility during rolling deploys**: If unpatched Stage 0 replicas and patched Stage 1 replicas run simultaneously under the **same** `app_version`, an unpatched replica claiming a workflow started by Stage 1 will encounter `DBOS.patch-*` and fail that recovery attempt with `UnexpectedStep`. Drain or replace Stage 0 replicas promptly when rolling forward under a shared `app_version`.

Reference: [dbos::patch](https://docs.rs/dbos/latest/dbos/fn.patch.html), [dbos::deprecate_patch](https://docs.rs/dbos/latest/dbos/fn.deprecate_patch.html)
