---
title: Rate Limit Queues for External APIs
impact: MEDIUM
impactDescription: Prevents exceeding third-party API rate limits
tags: queue, rate-limit, api, throttling
---

## Rate Limit Queues for External APIs

A rate limit caps how many workflows a queue may start in a rolling period, globally across all processes. Use it
when a downstream API enforces a request quota — unlike a concurrency limit, it bounds start rate rather than
in-flight count.

**Incorrect (throttling by sleeping in the workflow):**

```java
@Workflow
public void callApi(String request) throws Exception {
  dbos.sleep(Duration.ofSeconds(1)); // guesswork; breaks down with multiple processes
  dbos.runStep(() -> rateLimitedApi(request), "callApi");
}
```

**Correct (rate limit on the queue):**

```java
import dev.dbos.transact.workflow.QueueOptions;

// At most 100 workflow starts per 60 seconds across the entire application
dbos.registerQueue("api-queue",
    new QueueOptions().withRateLimit(100, 60, TimeUnit.SECONDS));

// Equivalent with a Duration
dbos.registerQueue("api-queue",
    new QueueOptions().withRateLimit(100, Duration.ofSeconds(60)));

// Combine with a concurrency limit
dbos.registerQueue("api-queue",
    new QueueOptions().withRateLimit(100, Duration.ofSeconds(60)).withWorkerConcurrency(5));
```

Behavior:

- The limit counts workflow *starts* in a rolling window; long-running workflows do not hold a slot
- Limits are enforced globally through the system database, so they hold no matter how many processes are running
- Workflows above the limit stay `ENQUEUED` and start as the window opens up
- A rate limit is set and cleared as a pair: pass `null` for both parameters
  (`new QueueOptions().withRateLimit(null, null)`) to clear one. Registering a queue with only the max or only the
  period set throws `IllegalArgumentException`. On `updateQueue`, `withRateLimitMax(Integer)` or
  `withRateLimitPeriod(Duration)` changes one half of a stored limit and keeps the other; on a queue with no stored
  limit, half a limit throws
- Rate limits and concurrency limits compose
- `withPartitionRateLimit(max, period)` applies a rate limit per partition key, alongside rather than instead of the
  queue-wide one ([queue-partitioning.md](queue-partitioning.md))

Common use cases:

- LLM API rate limiting (OpenAI, Anthropic, etc.)
- Third-party API throttling
- Preventing database overload

### Reconfiguring at Runtime

Because queue configuration lives in the system database, you can change a queue's rate limit at runtime without
redeploying ([queue-management.md](queue-management.md)):

```java
dbos.updateQueue("api-queue", new QueueOptions().withRateLimit(25, Duration.ofSeconds(30)));

// Change only the max; the stored period carries over
dbos.updateQueue("api-queue", new QueueOptions().withRateLimitMax(50));

// Or remove the limit entirely
dbos.updateQueue("api-queue", new QueueOptions().withRateLimit(null, null));
```

Reference: [Rate Limiting](https://docs.dbos.dev/java/tutorials/queue-tutorial#rate-limiting)
