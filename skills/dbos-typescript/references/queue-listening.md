---
title: Control Which Queues a Worker Listens To
impact: HIGH
impactDescription: Enables heterogeneous worker pools
tags: queue, listen, worker, process, configuration
---

## Control Which Queues a Worker Listens To

Configure `listenQueues` in DBOS configuration to make a process only dequeue from specific queues. This enables heterogeneous worker pools.

**Incorrect (all workers process all queues):**

```typescript
import { DBOS } from "@dbos-inc/dbos-sdk";

// Every worker processes both CPU and GPU tasks
// GPU tasks on CPU workers will fail or be slow!
DBOS.setConfig({
  name: "my-app",
  systemDatabaseUrl: process.env.DBOS_SYSTEM_DATABASE_URL,
});
await DBOS.launch();
await DBOS.registerQueue("cpu_queue");
await DBOS.registerQueue("gpu_queue");
```

**Correct (selective queue listening):**

```typescript
import { DBOS } from "@dbos-inc/dbos-sdk";

async function main() {
  const workerType = process.env.WORKER_TYPE; // "cpu" or "gpu"

  const config: any = {
    name: "my-app",
    systemDatabaseUrl: process.env.DBOS_SYSTEM_DATABASE_URL,
  };

  if (workerType === "gpu") {
    config.listenQueues = ["gpu_queue"];
  } else if (workerType === "cpu") {
    config.listenQueues = ["cpu_queue"];
  }

  DBOS.setConfig(config);
  await DBOS.launch();
  await DBOS.registerQueue("cpu_queue");
  await DBOS.registerQueue("gpu_queue");
}
```

`listenQueues` entries can be either a `WorkflowQueue` instance or a queue name (in-memory or database-backed). Names that don't match any queue at launch are deferred until a database-backed queue is registered with that name.

`listenQueues` only controls dequeuing. A CPU worker can still enqueue tasks onto the GPU queue:

```typescript
// From a CPU worker, enqueue onto the GPU queue
await DBOS.startWorkflow(gpuTask, { queueName: "gpu_queue" })("data");
```

Reference: [Explicit Queue Listening](https://docs.dbos.dev/typescript/tutorials/queue-tutorial#explicit-queue-listening)
