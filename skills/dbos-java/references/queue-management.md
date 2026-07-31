---
title: Reconfigure Queues at Runtime
impact: MEDIUM
impactDescription: Adjusts flow control without redeploying or restarting workers
tags: queue, management, updateQueue, conflict-resolution, operations
---

## Reconfigure Queues at Runtime

Queue configuration lives in the system database, so limits can be changed while the application runs. Use
`updateQueue` to modify only the fields you pass; absent fields keep their current values.

**Incorrect (redeploying to change a limit):**

```java
// Changing the registerQueue call and restarting every worker is
// unnecessary — and this call would only take effect on the next deploy
dbos.registerQueue("email-queue", QueueOptions.setConcurrency(20));
```

**Correct (update the live configuration):**

```java
// Change only the concurrency; the rate limit and other fields are untouched
dbos.updateQueue("email-queue", QueueOptions.setConcurrency(20));

// Change the rate limit
dbos.updateQueue("email-queue", QueueOptions.setRateLimit(25, 30, TimeUnit.SECONDS));

// Inspect what is registered
Optional<Queue> queue = dbos.findQueue("email-queue");
List<Queue> queues = dbos.listQueues();

// Remove a queue (drain or cancel its workflows first)
boolean deleted = dbos.deleteQueue("email-queue");
```

Workers pick up new configuration on their next polling iteration.

Startup conflicts: if your application calls `registerQueue` on every start, the next process to boot can overwrite
runtime changes. Control this with `QueueConflictResolution`:

- `UPDATE_IF_LATEST_VERSION` (default for `dbos.registerQueue`) — overwrite only if this executor runs the latest
  registered application version
- `NEVER_UPDATE` — leave existing configuration alone, preserving runtime changes
- `ALWAYS_UPDATE` (default for `DBOSClient.registerQueue`) — always overwrite

```java
dbos.registerQueue("email-queue",
    QueueOptions.setConcurrency(10),
    QueueConflictResolution.NEVER_UPDATE);
```

Field semantics: each `QueueOptions` field is tri-state. Absent means "leave unchanged", a value sets it, and
`null` clears it (for example `QueueOptions.setConcurrency(null)` removes the concurrency limit). The `set*`/`and*`
helpers build these values; `Field.absent()` and `Field.of(value)` are available for direct construction.

Deleting a queue leaves its enqueued workflows unrunnable — they resume only if a queue with the same name is
registered later, which is rarely intended. Cancel or drain pending workflows before deleting.

The same management methods are available on `DBOSClient` for admin tooling that runs outside the application.

Reference: [Reconfiguring Queues at Runtime](https://docs.dbos.dev/java/tutorials/queue-tutorial#reconfiguring-queues-at-runtime)
