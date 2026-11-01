---
title: Configure Step Retries, Timeouts, and Preemption
impact: HIGH
impactDescription: Handles transient failures with exponential backoff inside a single durable checkpoint
tags: step, retries, backoff, StepOptions, should_retry, timeout, preemptible
---

## Configure Step Retries, Timeouts, and Preemption

Use `dbos::step_with` and `StepOptions` to retry transient failures with exponential backoff instead of writing manual retry loops in workflow code. All retry attempts share a single step checkpoint: DBOS checks the checkpoint once before attempt 1 and records only the final outcome after retries succeed, `should_retry` declines, or `max_attempts` is exhausted.

**Incorrect (manual retry loop in workflow body burning step IDs and sleeping non-durably):**

```rust
use std::time::Duration;

async fn call_flaky_api(endpoint: String) -> dbos::Result<String> {
    for attempt in 1..=3 {
        match dbos::step("http_call", || fetch_url(&endpoint)).await {
            Ok(body) => return Ok(body),
            Err(err) if attempt < 3 => {
                // Non-durable sleep, and each iteration records a separate failed step row!
                tokio::time::sleep(Duration::from_secs(attempt)).await;
            }
            Err(err) => return Err(err),
        }
    }
    unreachable!()
}
```

**Correct (declarative retries, per-attempt timeout, and selective retry predicate with step_with):**

```rust
use std::time::Duration;
use dbos::{Error, StepOptions};
use serde::{Deserialize, Serialize};

#[derive(Debug, thiserror::Error, Serialize, Deserialize)]
enum ApiError {
    #[error("rate limited or transient HTTP {status}")]
    Transient { status: u16 },
    #[error("permanent client error HTTP {status}: {body}")]
    Permanent { status: u16, body: String },
}

async fn fetch_url(endpoint: &str) -> dbos::Result<String, ApiError> {
    if let Some(status) = dbos::step_status() {
        tracing::info!(
            attempt = status.current_attempt(),
            max = status.max_attempts(),
            "calling {endpoint}"
        );
    }
    Ok(format!("response from {endpoint}"))
}

async fn call_flaky_api(endpoint: String) -> dbos::Result<String, ApiError> {
    let options = StepOptions {
        max_attempts: 4,
        interval: Duration::from_secs(1),
        backoff_rate: 2.0,
        max_interval: Duration::from_secs(30),
        timeout: Some(Duration::from_secs(10)),
        ..StepOptions::default()
    }
    .should_retry(|err| match err {
        Error::Application(ApiError::Permanent { .. }) => false,
        Error::Application(ApiError::Transient { .. }) | Error::StepTimeout { .. } => true,
        _ => true,
    });

    dbos::step_with("http_call", options, || fetch_url(&endpoint)).await
}
```

### `StepOptions` Defaults and Failure Semantics

- `max_attempts`: Total attempts including the first try (default `1`, meaning no retries; `0` is treated as `1`).
- `interval`: Base delay after attempt 1 fails (default `1s`).
- `backoff_rate`: Multiplier per subsequent failure, `interval * backoff_rate.powi(failures)` (default `2.0`, clamped safely to `max_interval`).
- `max_interval`: Upper bound on backoff delay between attempts (default `3600s` / 1 hour).
- `timeout`: Bounds **each individual attempt** (default `None`; backoff sleep is not charged against it). Expiry cancels the attempt's `dbos::cancellation_token()`, drops the future, and yields `Error::StepTimeout { step, timeout }`, which is retryable when attempts remain.
- `preemptible`: When `true` (default `false`), polls the enclosing workflow's status while the step runs; if cancelled externally, aborts the step immediately without recording an outcome and returns `Error::WorkflowCancelled`.
- `.should_retry(|err: &Error<E>| -> bool)`: Synchronous predicate evaluated before sleeping. Returning `false` stops immediately and records/returns that attempt's unwrapped error.
- **Exhaustion error shape**: When `max_attempts == 1` or `should_retry` returns `false`, `step_with` returns that single `Error<E>` directly. When `max_attempts > 1` and every attempt fails, `step_with` checkpoints and returns `Error::MaxStepRetriesExceeded { step, attempts, errors }` containing all attempt errors in chronological order.
- **Attempt introspection**: Inside a running step body, `dbos::step_status()` returns `Some(StepStatus)` with `step_id()` (0-based), `current_attempt()` (1-based), and `max_attempts()`.

Reference: [dbos::step_with](https://docs.rs/dbos/latest/dbos/fn.step_with.html), [dbos::StepOptions](https://docs.rs/dbos/latest/dbos/struct.StepOptions.html), [dbos::StepStatus](https://docs.rs/dbos/latest/dbos/struct.StepStatus.html)
