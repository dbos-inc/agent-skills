---
title: Enqueue Workflows from External Applications
impact: MEDIUM
impactDescription: Enables external services to submit work to DBOS queues
tags: client, enqueue, external, queue
---

## Enqueue Workflows from External Applications

Use `client.enqueue()` to submit workflows from outside your DBOS application. Since `DBOSClient` runs externally, workflow and queue metadata must be specified explicitly.

**Incorrect (trying to use DBOS.startWorkflow from external code):**

```typescript
// DBOS.startWorkflow requires a full DBOS setup
await DBOS.startWorkflow(processTask, { queueName: "myQueue" })("data");
```

**Correct (using DBOSClient.enqueue):**

```typescript
import { DBOSClient } from "@dbos-inc/dbos-sdk";

const client = await DBOSClient.create({
  systemDatabaseUrl: process.env.DBOS_SYSTEM_DATABASE_URL!,
  // The application that owns and runs the workflows (set if apps share a system database)
  applicationName: "my-app",
});

// Optionally register the queue from the client (persists to system database)
await client.registerQueue("task_queue", { globalConcurrency: 10 });

// Basic enqueue
const handle = await client.enqueue(
  {
    workflowName: "processTask",
    queueName: "task_queue",
  },
  "task-data"
);

// Wait for the result
const result = await handle.getResult();
```

The queue must be registered (with `DBOS.registerQueue` or `client.registerQueue`) for the workflow to run. A workflow enqueued on an unregistered queue is durably recorded as `ENQUEUED` and stays there until the queue is registered and a worker becomes available.

**Type-safe enqueue:**

```typescript
// Import or declare the workflow type
declare class Tasks {
  static processTask(data: string): Promise<string>;
}

const handle = await client.enqueue<typeof Tasks.processTask>(
  {
    workflowName: "processTask",
    workflowClassName: "Tasks",
    queueName: "task_queue",
  },
  "task-data"
);

// TypeScript infers the result type
const result = await handle.getResult(); // type: string
```

**Enqueue options:**
- `workflowName` (required): Name of the workflow function
- `queueName` (required): Name of the queue
- `workflowClassName`: Class name if the workflow is a class method
- `workflowConfigName`: Instance name if using `ConfiguredInstance`
- `workflowID`: Custom workflow ID
- `workflowTimeoutMS`: Timeout in milliseconds
- `deduplicationID`: Prevent duplicate enqueues
- `priority`: Queue priority (lower = higher priority)
- `delaySeconds`: Delay before becoming eligible for execution
- `queuePartitionKey`: Partition key for partitioned queues
- `appVersion`: Pin the workflow to a specific application version. If unset, the workflow is only dequeued by an executor running the application's latest registered version, and takes that executor's version when dequeued
- `duplicationPolicy`: How to handle a `deduplicationID` collision. `'reject'` (default) throws `DBOSQueueDuplicatedError`; `'return-existing'` attaches to the existing workflow and returns its handle (singleton pattern — requires `deduplicationID`)
- `applicationName`: Application that owns and runs the workflow (defaults to the client's `applicationName`)
- `serializationType`: Serialization strategy for workflow arguments (`"portable"` for cross-language interop, otherwise the configured serializer is used)

**Singleton workflow example (`return-existing`):**

```typescript
const handle = await client.enqueue(
  {
    workflowName: "processTask",
    queueName: "task_queue",
    deduplicationID: "singleton",
    duplicationPolicy: "return-existing",
  },
  "task-data"
);
// If a workflow with deduplicationID "singleton" is already enqueued or
// running on this queue, handle resolves to that workflow's result instead
// of throwing.
```

**Cross-language enqueue (`serializationType: "portable"`):**

```typescript
await client.enqueue(
  {
    workflowName: "processOrder",
    queueName: "orders",
    serializationType: "portable",  // Python/Java/Go workers can read these args
  },
  "order-123"
);
```

### Enqueueing Atomically With Your Own Writes

`client.enqueueInTransaction` performs the enqueue inside a transaction you own, so the workflow is enqueued if and only if your database writes commit. Pass a `node-postgres` `Client` or `PoolClient` with an open transaction, **connected to the DBOS system database**:

```typescript
import { Client } from "pg";

declare class Orders {
  static processOrder(orderId: string): Promise<void>;
}

const orderId = "order-123";
const pg = new Client({ connectionString: process.env.DBOS_SYSTEM_DATABASE_URL });
await pg.connect();
try {
  await pg.query("BEGIN");
  await pg.query("INSERT INTO orders (id, status) VALUES ($1, 'new')", [orderId]);
  const handle = await client.enqueueInTransaction<typeof Orders.processOrder>(
    pg,
    { workflowName: "processOrder", workflowClassName: "Orders", queueName: "orders" },
    orderId,
  );
  await pg.query("COMMIT"); // The workflow does not exist until this commits
  await handle.getResult(); // Only call getResult() after the commit
} catch (e) {
  await pg.query("ROLLBACK"); // Neither the row nor the workflow is created
  throw e;
} finally {
  await pg.end();
}
```

- You own the transaction: DBOS never begins, commits, rolls back, or retries it
- `duplicationPolicy: 'return-existing'` is not supported (throws)
- `client.enqueuePortableInTransaction(pg, options, positionalArgs, namedArgs?)` is the portable-serialization variant (for targets with named arguments, e.g. Python kwargs); `client.enqueuePortable(options, positionalArgs, namedArgs?)` is the non-transactional one
- `client.sendInTransaction(pg, destinationID, message, topic?, idempotencyKey?)` sends a message atomically in the same way (see `comm-messages.md`)

Always call `client.destroy()` when done.

Reference: [DBOS Client Enqueue](https://docs.dbos.dev/typescript/reference/client#enqueue)
