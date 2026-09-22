---
title: Restrict Which Queues a Process Listens To
impact: MEDIUM
impactDescription: Routes workloads to the right workers in heterogeneous deployments
tags: queue, listening, workers, routing, configuration
---

## Restrict Which Queues a Process Listens To

By default a DBOS process dequeues from every queue registered in its system database. Use `withListenQueues` to
restrict a process to specific queues when workers are heterogeneous — GPU versus CPU machines, or a worker pool
dedicated to slow batch jobs.

**Incorrect (relying on every worker to handle every queue):**

```java
// Every process listens to gpuQueue, so GPU jobs land on CPU-only machines
var config = DBOSConfig.defaultsFromEnv("my-app").withAppVersion("0.1.0");
```

**Correct (each worker type listens to its own queue):**

```java
var workerType = System.getenv("WORKER_TYPE"); // "cpu" or "gpu"

var config = DBOSConfig.defaultsFromEnv("my-app").withAppVersion("0.1.0");
if ("gpu".equals(workerType)) {
  config = config.withListenQueues("gpuQueue");
} else {
  config = config.withListenQueues("cpuQueue");
}

DBOS dbos = new DBOS(config);
// register workflow classes...
dbos.launch();

dbos.registerQueue("cpuQueue", QueueOptions.setWorkerConcurrency(8));
dbos.registerQueue("gpuQueue", QueueOptions.setWorkerConcurrency(1));
```

Notes:

- `withListenQueues` controls only dequeuing, not enqueueing — any process may enqueue onto any queue, so a CPU
  worker can hand GPU work to GPU workers
- Overloads accept queue names as `String` or `QueueName` (the `Queue` overloads are deprecated for removal);
  `withListenQueue(...)` adds a single queue
- A queue with no listening process accumulates `ENQUEUED` workflows indefinitely — make sure some deployment
  listens to every queue you enqueue onto
- With the Spring Boot starter, set `dbos.listen-queues` in `application.yaml`
- Recovery goes through the queues: a recovered `PENDING` workflow is re-enqueued onto its own queue (or onto the
  internal queue if it had none), and whichever process listens to that queue runs it — not necessarily the process
  that recovered it. Every process always polls the internal queue, whatever `withListenQueues` says

Reference: [Explicit Queue Listening](https://docs.dbos.dev/java/tutorials/queue-tutorial#explicit-queue-listening)
