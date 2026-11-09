---
title: Use Durable Sleep for Delayed Execution
impact: MEDIUM
impactDescription: A non-durable sleep restarts from zero after every crash
tags: pattern, sleep, delay, timer
---

## Use Durable Sleep for Delayed Execution

`dbos::sleep(duration)` records its wake time, so a recovered workflow wakes on the original schedule; it can sleep for days. `tokio::time::sleep` in a workflow body isn't recorded and runs again in full on recovery. Outside a workflow, `dbos::sleep` is a plain sleep.

**Incorrect (tokio::time::sleep in a workflow):**

```rust
async fn reminder_workflow(input: Reminder) -> dbos::Result<String> {
    // WRONG: a crash an hour in restarts the full wait.
    tokio::time::sleep(input.time_to_sleep).await;
    step("run_task", || run_task(&input.task)).await
}
```

**Correct (dbos::sleep):**

```rust
#[derive(Serialize, Deserialize)]
struct Reminder {
    time_to_sleep: Duration,
    task: String,
}

async fn reminder_workflow(input: Reminder) -> dbos::Result<String> {
    // Sleep for the specified duration.
    dbos::sleep(input.time_to_sleep).await?;
    // Execute the task after sleeping.
    step("run_task", || run_task(&input.task)).await
}
```
