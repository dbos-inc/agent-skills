---
title: Manage Database-Backed Queues at Runtime
impact: HIGH
impactDescription: Inspect, reconfigure, and delete queues without redeploying
tags: queue, retrieve, delete, list, reconfigure, runtime
---

## Manage Database-Backed Queues at Runtime

Queue configuration lives in the system database, so any DBOS process or `DBOSClient` connected to the same database can inspect and reconfigure queues without restarts or redeploys.

**Incorrect (redeploying just to change a limit):**

```typescript
// Old approach: hardcoded in source, ship a new deploy to change.
await DBOS.registerQueue("email", { concurrency: 10 });
```

**Correct (reconfigure at runtime):**

```typescript
// From an admin tool or a running DBOS process - no redeploy needed.
const queue = await DBOS.retrieveQueue("email");
if (queue !== null) {
  await queue.setConcurrency(50);
  await queue.setRateLimit({ limitPerPeriod: 500, periodSec: 60 });
}
```

Workers pick up the new configuration on their next polling iteration.

### Retrieving and Listing Queues

```typescript
const queue = await DBOS.retrieveQueue("email"); // null if not registered
if (queue !== null) {
  console.log(await queue.getConcurrency());
}

const queues = await DBOS.listQueues();
for (const q of queues) {
  console.log(q.name, q.concurrency);
}
```

Use the `get*` methods (`getConcurrency()`, `getRateLimit()`, etc.) to read the latest value from the database; the cached fields on the `WorkflowQueue` object may be stale if another process has reconfigured the queue.

### All Reconfiguration Methods

```typescript
await queue.setConcurrency(50);            // or undefined to remove
await queue.setWorkerConcurrency(5);
await queue.setRateLimit({ limitPerPeriod: 500, periodSec: 60 });
await queue.setPriorityEnabled(true);
await queue.setPartitionQueue(false);
await queue.setMinPollingIntervalMs(2000);
```

The `set` methods may only be called on a queue returned from `DBOS.registerQueue`, `DBOS.retrieveQueue`, or the equivalent `DBOSClient` methods. Calling them on a queue created with the legacy `new WorkflowQueue(...)` constructor throws an error.

**Warning:** If your application calls `DBOS.registerQueue` on startup, the next process to launch can overwrite settings you applied via `set` methods. Either update the `registerQueue` call to match, or pass `onConflict: 'never_update'` to preserve runtime changes.

### Deleting a Queue

```typescript
await DBOS.deleteQueue("email");
```

No-op if the queue does not exist.

**Warning:** Workflows already enqueued on a deleted queue can no longer be dequeued, executed, or recovered. Cancel or drain pending workflows on the queue before deleting it.

### From a DBOSClient

The same methods are available on `DBOSClient` for external services and admin tools:

```typescript
await client.registerQueue("email", { concurrency: 10, onConflict: "always_update" });
await client.retrieveQueue("email");
await client.listQueues();
await client.deleteQueue("email");
```

`onConflict: 'update_if_latest_version'` is **not** supported on the client (clients have no application version). The client's `onConflict` default is `'always_update'`.

Reference: [Queues Reference](https://docs.dbos.dev/typescript/reference/queues)
