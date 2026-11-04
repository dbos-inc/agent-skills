---
title: Rate Limit Queue Execution
impact: HIGH
impactDescription: Bursts exceed downstream rate limits
tags: queue, rate-limit, throttle, llm
---

## Rate Limit Queue Execution

Set `rate_limit: Some(RateLimit { limit, period })` in `QueueOptions`: at most `limit` workflows start per `period`, across every process. Use it for rate-limited APIs such as LLM providers. `partition_rate_limit` applies one per partition key.

**Incorrect (a sleep in each workflow):**

```rust
async fn call_llm_bad(prompt: String) -> dbos::Result<String> {
    // WRONG: a hand-rolled delay only paces this one workflow, not the fleet.
    tokio::time::sleep(Duration::from_millis(600)).await;
    step("call_llm", || async { Ok(prompt.clone()) }).await
}
```

**Correct (a rate-limited queue):**

```rust
async fn llm_queue(dbos: &DBOS) -> dbos::Result<()> {
    // At most 100 starts per minute, across every process.
    let options = QueueOptions {
        rate_limit: Some(RateLimit {
            limit: 100,
            period: Duration::from_secs(60),
        }),
        ..QueueOptions::default()
    };
    dbos.register_queue("llm_queue", options, QueueConflict::UpdateIfLatestVersion)
        .await?;
    Ok(())
}
```
