---
title: Use dbos::sleep Instead of tokio::time::sleep in Workflows
impact: MEDIUM
impactDescription: Persists the wake-up timestamp in PostgreSQL so workflows sleep only the remaining duration after restarts
tags: pattern, sleep, timer, durable, recovery
---

## Use dbos::sleep Instead of tokio::time::sleep in Workflows

Call `dbos::sleep(duration).await?` inside workflows whenever execution needs to pause for seconds, hours, or days. `tokio::time::sleep` holds its deadline only in process memory, so a restart resets the entire wait from zero; `dbos::sleep` checkpoints the target wake-up timestamp in PostgreSQL **before** waiting and sleeps only the remaining duration (or returns immediately if the deadline already passed) when recovered.

**Incorrect (tokio::time::sleep in workflow body resets on every restart):**

```rust
use std::time::Duration;

async fn trial_reminder(user_id: String) -> dbos::Result<()> {
    dbos::step("send_welcome_email", || send_email(&user_id, "welcome")).await?;

    // Resets to a fresh 7 days every time the server deploys or restarts!
    tokio::time::sleep(Duration::from_secs(7 * 24 * 3600)).await;

    dbos::step("send_followup_email", || send_email(&user_id, "followup")).await
}
```

**Correct (durable dbos::sleep survives deploys and crashes):**

```rust
use std::time::Duration;

async fn send_email(user_id: &str, template: &str) -> dbos::Result<()> {
    let _ = (user_id, template);
    Ok(())
}

async fn trial_reminder(user_id: String) -> dbos::Result<()> {
    dbos::step("send_welcome_email", || send_email(&user_id, "welcome")).await?;

    // Checkpoints `wake_at = now + 7 days` in Postgres; restarts sleep only what remains
    dbos::sleep(Duration::from_secs(7 * 24 * 3600)).await?;

    dbos::step("send_followup_email", || send_email(&user_id, "followup")).await
}
```

### Details and When to Use Alternative Delays

- **Postponing the start of a workflow**: If a workflow does no work before sleeping, enqueue it with `Enqueue { delay: Some(duration), ..Enqueue::new("queue") }` instead so no worker task is held open during the wait (see [queue-delay.md](queue-delay.md)).
- **Inside a step or unit test**: Calling `dbos::sleep` inside a running `dbos::step` body or outside any workflow falls back to a plain `tokio::time::sleep(duration)` with no database write (`tokio::time::sleep` directly inside a step is also fine for brief intra-step pauses).
- **Bounds**: `Duration::ZERO` is valid and records an immediate checkpoint inside a workflow. Durations exceeding `i64::MAX` milliseconds fail immediately with `Error::InvalidArgument`.

Reference: [dbos::sleep](https://docs.rs/dbos/latest/dbos/fn.sleep.html)
