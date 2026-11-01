---
title: Throttle Workflow Starts with Queue Rate Limiting
impact: MEDIUM
impactDescription: Caps how many queued workflows can start within a sliding database-clock time window
tags: queue, rate-limit, RateLimit, throttling, api-quota
---

## Throttle Workflow Starts with Queue Rate Limiting

Configure `rate_limit: Some(RateLimit { limit, period })` on `QueueOptions` to bound how many workflows from a queue may **start** across the entire fleet within any trailing `period` window. Unlike `concurrency` (which bounds how many workflows are running right now), `rate_limit` bounds the **start rate** even when individual workflows finish in milliseconds.

**Incorrect (passing zero/negative limits or assuming rate_limit caps in-flight count):**

```rust
use std::time::Duration;
use dbos::{DBOS, QueueConflict, QueueOptions, RateLimit};

async fn bad_rate_limit(dbos: &DBOS) -> dbos::Result<()> {
    // Rejected with `Error::InvalidArgument`: both `limit` (> 0) and `period` (> ZERO) must be positive!
    dbos.register_queue(
        "stripe-webhooks",
        QueueOptions {
            rate_limit: Some(RateLimit {
                limit: 0,
                period: Duration::ZERO,
            }),
            ..QueueOptions::default()
        },
        QueueConflict::UpdateIfLatestVersion,
    )
    .await?;
    Ok(())
}
```

**Correct (combining fleet-wide rate_limit with concurrency protection):**

```rust
use std::time::Duration;
use dbos::{DBOS, QueueConflict, QueueOptions, RateLimit};

async fn register_rate_limited_queue(dbos: &DBOS) -> dbos::Result<()> {
    // Starts at most 100 workflows per 60 seconds fleet-wide, with at most 10 in flight at once
    dbos.register_queue(
        "stripe-webhooks",
        QueueOptions {
            rate_limit: Some(RateLimit {
                limit: 100,
                period: Duration::from_secs(60),
            }),
            concurrency: Some(10),
            worker_concurrency: Some(5),
            ..QueueOptions::default()
        },
        QueueConflict::UpdateIfLatestVersion,
    )
    .await?;
    Ok(())
}
```

### Rate Limit Semantics

- **Database-clock sliding window**: Every dequeue counts workflows on the queue whose `started_at` falls within `now - period` on the PostgreSQL clock, so clock skew across worker hosts never distorts the quota.
- **Combine with `concurrency` for slow downstream outages**: `RateLimit` alone caps starts per window; if downstream latency spikes from 50 ms to 3 minutes, new windows still admit another `limit` workflows. Pairing `rate_limit` with `concurrency` or `worker_concurrency` caps both arrival rate and simultaneous load.
- **Per-partition rate limits**: Set `partition_rate_limit: Some(RateLimit { limit, period })` (alongside `Enqueue { partition_key: Some(tenant), .. }`) to enforce per-tenant or per-destination rate quotas independently of the queue-wide `rate_limit` (see [queue-partitioning.md](queue-partitioning.md)).

Reference: [dbos::RateLimit](https://docs.rs/dbos/latest/dbos/sysdb/types/struct.RateLimit.html), [dbos::QueueOptions::rate_limit](https://docs.rs/dbos/latest/dbos/struct.QueueOptions.html#structfield.rate_limit)
