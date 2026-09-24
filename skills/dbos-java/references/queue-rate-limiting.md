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
    QueueOptions.setRateLimit(100, 60, TimeUnit.SECONDS));

// Equivalent with a Duration
dbos.registerQueue("api-queue",
    QueueOptions.setRateLimit(100, Duration.ofSeconds(60)));

// Combine with a concurrency limit
dbos.registerQueue("api-queue",
    QueueOptions.setRateLimit(100, Duration.ofSeconds(60)).andWorkerConcurrency(5));
```

Behavior:

- The limit counts workflow *starts* in a rolling window; long-running workflows do not hold a slot
- Limits are enforced globally through the system database, so they hold no matter how many processes are running
- Workflows above the limit stay `ENQUEUED` and start as the window opens up
- A rate limit is set and cleared as a pair: pass `null` for both parameters
  (`QueueOptions.setRateLimit(null, null)`) to clear one. Registering a queue with only the max or only the period
  set throws `IllegalArgumentException`. An update may change just the max or just the period of an existing limit,
  but one that would leave only one of them set (such as `setRateLimit(5, null)` on a queue with no limit) throws
- Rate limits and concurrency limits compose
- `andPartitionRateLimit(max, period)` applies a rate limit per partition key, alongside rather than instead of the
  queue-wide one ([queue-partitioning.md](queue-partitioning.md))

Common use cases:

- LLM API rate limiting (OpenAI, Anthropic, etc.)
- Third-party API throttling
- Preventing database overload

### Reconfiguring at Runtime

Because queue configuration lives in the system database, you can change a queue's rate limit at runtime without
redeploying ([queue-management.md](queue-management.md)):

```java
dbos.updateQueue("api-queue", QueueOptions.setRateLimit(25, Duration.ofSeconds(30)));

// Or remove the limit entirely
dbos.updateQueue("api-queue", QueueOptions.setRateLimit(null, null));
```

Reference: [Rate Limiting](https://docs.dbos.dev/java/tutorials/queue-tutorial#rate-limiting)
