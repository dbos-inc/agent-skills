---
title: Configure Step Retries for Transient Failures
impact: HIGH
impactDescription: Transient failures fail the workflow when a retry would have succeeded
tags: step, retries, backoff, timeout, should_retry, preemptible
---

## Configure Step Retries for Transient Failures

Configure retries with `dbos::step_with(name, StepOptions { .. }, f)`. `max_attempts` defaults to 1 (no retries); `interval` 1s; `backoff_rate` 2.0; `max_interval` 1h. `.should_retry(|e| ...)` declines errors that won't get better. `timeout` bounds each attempt (it fails with `Error::StepTimeout` and counts as an attempt). With `max_attempts` above 1, a step that fails every attempt returns `Error::MaxStepRetriesExceeded` with every attempt's error; with the default of 1, its own error comes back unwrapped.

**Incorrect (a hand-written retry loop in the workflow):**

```rust
async fn fetch_workflow(url: String) -> dbos::Result<String, FetchError> {
    // WRONG: each attempt is its own step, and a crash loses the loop's place.
    for attempt in 0..10 {
        if let Ok(body) = step("fetch_page", || fetch_page(&url)).await {
            return Ok(body);
        }
    }
    Err(FetchError::Request("gave up".to_owned()))?
}
```

**Correct (StepOptions with a retry predicate):**

```rust
#[derive(Debug, thiserror::Error, Serialize, Deserialize)]
enum FetchError {
    #[error("request failed: {0}")]
    Request(String),
    #[error("the server answered {0}")]
    Status(u16),
}

async fn fetch_page(url: &str) -> dbos::Result<String, FetchError> {
    let response = reqwest::get(url)
        .await
        .map_err(|error| FetchError::Request(error.to_string()))?;
    if !response.status().is_success() {
        Err(FetchError::Status(response.status().as_u16()))?;
    }
    let body = response
        .text()
        .await
        .map_err(|error| FetchError::Request(error.to_string()))?;
    Ok(body)
}

async fn fetch_workflow(url: String) -> dbos::Result<String, FetchError> {
    let options = StepOptions {
        max_attempts: 10,
        interval: Duration::from_millis(500),
        backoff_rate: 2.0,
        max_interval: Duration::from_secs(30),
        ..StepOptions::default()
    }
    // Retry server errors and network failures, but not a 4xx.
    .should_retry(|error| match error {
        dbos::Error::Application(FetchError::Status(status)) => *status >= 500,
        _ => true,
    });
    step_with("fetch_page", options, || fetch_page(&url)).await
}
```

For work a dropped future can't stop (a `spawn_blocking` thread), watch `dbos::cancellation_token()` inside the step; it fires when DBOS abandons the attempt: a step timeout, a lost `select_step!` race, the workflow's timeout, shutdown, or the cancellation of a `preemptible` step. An ordinary cancel doesn't fire it: a step that isn't preemptible runs to its end. Set `preemptible: true` to stop the step as soon as its workflow is seen to be cancelled.

```rust
async fn blocking_step() -> dbos::Result<u64> {
    let token = dbos::cancellation_token();
    let total = tokio::task::spawn_blocking(move || {
        let mut total = 0;
        for chunk in 0..1_000_000u64 {
            if token.is_cancelled() {
                break;
            }
            total += chunk;
        }
        total
    })
    .await
    .expect("the blocking task panicked");
    Ok(total)
}
```
