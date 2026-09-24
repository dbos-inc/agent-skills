---
title: Manage Database-Backed Queues at Runtime
impact: HIGH
impactDescription: Change concurrency and rate limits without redeploying
tags: queue, registerQueue, retrieveQueue, listQueues, deleteQueue, runtime, reconfigure
---

## Manage Database-Backed Queues at Runtime

Queue configuration lives in the system database, so any DBOS process or `DBOSClient` connected to the same database can inspect and reconfigure queues without restarts or redeploys. Workers pick up the new configuration on their next polling iteration.

**Incorrect (redeploying just to change a limit):**

```typescript
// Hardcoded in source - ship a new deploy to change.
await DBOS.registerQueue("email", { globalConcurrency: 10 });
```

**Correct (reconfigure at runtime):**

```typescript
// From an admin tool or a running DBOS process - no redeploy needed.
const queue = await DBOS.retrieveQueue("email");
if (queue !== null) {
  await queue.setGlobalConcurrency(50);
  await queue.setWorkerConcurrency(5);
  await queue.setRateLimit({ limitPerPeriod: 500, periodSec: 60 });
  await queue.setRateLimit(undefined); // Remove the rate limit
}
```

### Retrieving and Listing Queues

```typescript
const queue = await DBOS.retrieveQueue("email"); // null if not registered
if (queue !== null) {
  console.log(await queue.getGlobalConcurrency());
}

// Lists this application's queues (pass applicationName to list others)
const queues = await DBOS.listQueues();
for (const q of queues) {
  console.log(q.name, q.concurrency, q.partitionConcurrency);
}
```

Use the `get*` methods to read the latest value from the database; the cached fields on the `WorkflowQueue` object (`concurrency`, `workerConcurrency`, `rateLimit`, `partitionConcurrency`, ...) may be stale if another process reconfigured the queue.

### All Get/Set Methods

```typescript
// Write through to the database. Pass undefined to remove a limit
// (except setMinPollingIntervalMs).
await queue.setGlobalConcurrency(50);
await queue.setWorkerConcurrency(5);
await queue.setRateLimit({ limitPerPeriod: 500, periodSec: 60 });
// Setting any partition limit partitions the queue (see note below)
await queue.setPartitionConcurrency(1);
await queue.setPartitionWorkerConcurrency(1);
await queue.setPartitionRateLimit({ limitPerPeriod: 10, periodSec: 60 });
await queue.setMinPollingIntervalMs(2000);

// Re-read from the database
await queue.getGlobalConcurrency();
await queue.getWorkerConcurrency();
await queue.getRateLimit();
await queue.getPartitionConcurrency();
await queue.getPartitionWorkerConcurrency();
await queue.getPartitionRateLimit();
await queue.getMinPollingIntervalMs();
```

Each `set` method validates the new value against the queue's other limits using the same rules as `registerQueue`.

Setting any partition limit partitions the queue; see `queue-partitioning.md` before doing this at runtime.

**Warning:** If your application calls `DBOS.registerQueue` on startup, the next process to launch can overwrite settings you applied via `set` methods. Either update the `registerQueue` call to match, or pass `onConflict: 'never_update'` to preserve runtime changes.

### Deleting a Queue

```typescript
await DBOS.deleteQueue("email");  // No-op if no queue with that name exists
```

**Warning:** Workflows already enqueued on a deleted queue can no longer be dequeued, executed, or recovered (unless a queue with the same name is registered again, which dequeues them - don't rely on this). Cancel or drain pending workflows on the queue before deleting it.

### From a DBOSClient

The same methods are available on `DBOSClient` for external services and admin tools:

```typescript
await client.registerQueue("email", { globalConcurrency: 10, onConflict: "always_update" });
await client.retrieveQueue("email");
await client.listQueues();
await client.deleteQueue("email");
```

`onConflict: 'update_if_latest_version'` is **not** supported on the client (clients have no application version). The client's `onConflict` default is `'always_update'`. Queues returned by the client can be reconfigured with `set` methods, but enqueue on them with `client.enqueue`.

Reference: [Queues Reference](https://docs.dbos.dev/typescript/reference/queues)
