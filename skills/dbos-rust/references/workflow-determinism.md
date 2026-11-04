---
title: Keep Workflows Deterministic
impact: CRITICAL
impactDescription: Non-deterministic workflows cannot recover correctly
tags: workflow, determinism, recovery, reliability
---

## Keep Workflows Deterministic

Workflow functions must be deterministic: given the same input and step results, they must make the same DBOS calls in the same order. Recovery runs the workflow function again and replays each completed step's recorded result. Move non-deterministic operations (the clock, random numbers, UUIDs, environment and file reads, network calls) into steps.

**Incorrect (non-deterministic workflow):**

```rust
async fn example_workflow(_: ()) -> dbos::Result<String> {
    // Random value in the workflow body breaks recovery!
    // On replay it may differ, so the workflow takes the other branch.
    if rand::random::<bool>() {
        step("step_one", step_one).await
    } else {
        step("step_two", step_two).await
    }
}
```

**Correct (non-determinism in a step):**

```rust
async fn example_workflow(_: ()) -> dbos::Result<String> {
    // The step's result is recorded, and replay uses the saved value.
    let choice = step("choose", || async { Ok(rand::random::<bool>()) }).await?;
    if choice {
        step("step_one", step_one).await
    } else {
        step("step_two", step_two).await
    }
}
```

A recovery that reaches a different step than the one recorded at that position is refused with `dbos::sysdb::Error::UnexpectedStep`; the workflow stays `PENDING`, recovery retries it, and after 100 attempts it is parked as `MAX_RECOVERY_ATTEMPTS_EXCEEDED`. DBOS compares step names, not inputs, so it can't catch every mistake. Iterate a `HashMap` in sorted order (or use a `BTreeMap`) before calling steps for its entries.
