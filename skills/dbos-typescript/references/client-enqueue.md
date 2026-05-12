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
  systemDatabaseUrl: process.env.DBOS_SYSTEM_DATABASE_URL,
});

// Optionally register the queue from the client (persists to system database)
await client.registerQueue("task_queue", { concurrency: 10 });

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

The queue does not need to exist when `enqueue` is called. If no queue with the given name has been registered, the workflow is still durably recorded as `ENQUEUED` and starts running once the queue is registered and a worker becomes available.

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
- `appVersion`: Pin the workflow to a specific application version
- `duplicationPolicy`: How to handle a `deduplicationID` collision. `'reject'` (default) throws `DBOSQueueDuplicatedError`; `'return-existing'` attaches to the existing workflow and returns its handle (singleton pattern — requires `deduplicationID`)
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

Always call `client.destroy()` when done.

Reference: [DBOS Client Enqueue](https://docs.dbos.dev/typescript/reference/client#enqueue)
