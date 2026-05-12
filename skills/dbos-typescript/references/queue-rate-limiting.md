---
title: Rate Limit Queue Execution
impact: HIGH
impactDescription: Prevents overwhelming external APIs with too many requests
tags: queue, rate-limit, throttle, api
---

## Rate Limit Queue Execution

Set rate limits on a queue to control how many workflows start in a given period. Rate limits are global across all DBOS processes.

**Incorrect (no rate limiting):**

```typescript
await DBOS.registerQueue("llm_tasks");
// Could send hundreds of requests per second to a rate-limited API
```

**Correct (rate-limited queue):**

```typescript
await DBOS.registerQueue("llm_tasks", {
  rateLimit: { limitPerPeriod: 50, periodSec: 30 },
});
```

This queue starts at most 50 workflows per 30 seconds.

**Combining rate limiting with concurrency:**

```typescript
// At most 5 concurrent and 50 per 30 seconds
await DBOS.registerQueue("api_tasks", {
  workerConcurrency: 5,
  rateLimit: { limitPerPeriod: 50, periodSec: 30 },
});
```

Common use cases:
- LLM API rate limiting (OpenAI, Anthropic, etc.)
- Third-party API throttling
- Preventing database overload

### Reconfiguring at Runtime

Because queue configuration lives in the system database, you can change a queue's rate limit at runtime without redeploying:

```typescript
const queue = await DBOS.retrieveQueue("llm_tasks");
if (queue !== null) {
  await queue.setRateLimit({ limitPerPeriod: 25, periodSec: 30 });

  // Or remove the limit entirely
  await queue.setRateLimit(undefined);
}
```

Reference: [Rate Limiting](https://docs.dbos.dev/typescript/tutorials/queue-tutorial#rate-limiting)
