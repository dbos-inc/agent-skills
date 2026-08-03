---
title: Control Which Queues a Worker Listens To
impact: HIGH
impactDescription: Enables heterogeneous worker pools
tags: queue, listen, worker, process, configuration
---

## Control Which Queues a Worker Listens To

Use `ListenQueues` to make a process only dequeue from specific queues. This enables heterogeneous worker pools.

**Incorrect (all workers process all queues):**

```go
cpuQueue, err := dbos.RegisterQueue(ctx, "cpu_queue")
gpuQueue, err := dbos.RegisterQueue(ctx, "gpu_queue")

// Every worker processes both CPU and GPU tasks
// GPU tasks on CPU workers will fail or be slow!
dbos.Launch(ctx)
```

**Correct (selective queue listening):**

```go
cpuQueue, err := dbos.RegisterQueue(ctx, "cpu_queue")
gpuQueue, err := dbos.RegisterQueue(ctx, "gpu_queue")

workerType := os.Getenv("WORKER_TYPE") // "cpu" or "gpu"

if workerType == "gpu" {
	dbos.ListenQueues(ctx, "gpu_queue")
} else if workerType == "cpu" {
	dbos.ListenQueues(ctx, "cpu_queue")
}

dbos.Launch(ctx)
```

`ListenQueues` takes queue names and each call replaces the entire listen set; calling it with no names restores the default of listening to every queue. Use `dbos.ListenedQueues(ctx)` to read the current set and modify it incrementally. `ListenQueues` only controls dequeuing. A CPU worker can still enqueue tasks onto the GPU queue:

```go
// From a CPU worker, enqueue onto the GPU queue
dbos.RunWorkflow(ctx, gpuTask, "data",
	dbos.WithQueue(gpuQueue),
)
```

Reference: [Listening to Specific Queues](https://docs.dbos.dev/golang/tutorials/queue-tutorial#listening-to-specific-queues)
