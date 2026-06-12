---
title: Enqueue Workflows from External Applications
impact: HIGH
impactDescription: Enables external services to submit work to DBOS queues
tags: client, enqueue, external, queue
---

## Enqueue Workflows from External Applications

Use `client.Enqueue()` to submit workflows from outside your DBOS application. Since the Client runs externally, workflow and queue metadata must be specified explicitly by name.

**Incorrect (trying to use RunWorkflow from external code):**

```go
// RunWorkflow requires a full DBOS context with registered workflows
dbos.RunWorkflow(ctx, processTask, "data", dbos.WithQueue("myQueue"))
```

**Correct (using Client.Enqueue):**

```go
client, err := dbos.NewClient(context.Background(), dbos.ClientConfig{
	DatabaseURL: os.Getenv("DBOS_SYSTEM_DATABASE_URL"),
})
if err != nil {
	log.Fatal(err)
}
defer client.Shutdown(10 * time.Second)

// Basic enqueue - specify workflow and queue by name
handle, err := client.Enqueue("task_queue", "processTask", "task-data")
if err != nil {
	log.Fatal(err)
}

// Wait for the result
result, err := handle.GetResult()
```

**Enqueue with options:**

```go
handle, err := client.Enqueue("task_queue", "processTask", "task-data",
	dbos.WithEnqueueWorkflowID("custom-id"),
	dbos.WithEnqueueDeduplicationID("unique-id"),
	dbos.WithEnqueuePriority(10),
	dbos.WithEnqueueTimeout(5*time.Minute),
	dbos.WithEnqueueQueuePartitionKey("user-123"),
	dbos.WithEnqueueApplicationVersion("2.0.0"),
)
```

Enqueue options:
- `WithEnqueueWorkflowID`: Custom workflow ID
- `WithEnqueueDeduplicationID`: Prevent duplicate enqueues
- `WithEnqueueDeduplicationPolicy`: How a colliding deduplication ID is handled — `DeduplicationPolicyReject` (default, returns a `QueueDeduplicated` error) or `DeduplicationPolicyReturnExisting` (returns a handle to the existing workflow)
- `WithEnqueuePriority`: Queue priority (lower = higher priority)
- `WithEnqueueTimeout`: Workflow timeout
- `WithEnqueueQueuePartitionKey`: Partition key for partitioned queues
- `WithEnqueueApplicationVersion`: Override application version
- `WithEnqueueDelay`: Delay execution; the workflow stays in `DELAYED` status until the delay expires
- `WithEnqueueClassName`: Class/namespace name, required when enqueueing to Python, TypeScript, or Java targets
- `WithEnqueueConfigName`: Config/instance name, required when the target workflow is registered on a configured instance (Go's `WithInstance`, Python/TypeScript/Java class instances)
- `WithEnqueueAuthenticatedUser`, `WithEnqueueAssumedRole`, `WithEnqueueAuthenticatedRoles`: Attach authentication metadata

The workflow name must match the registered name or custom name set with `WithWorkflowName` during registration.

Always call `client.Shutdown()` when done.

Reference: [DBOS Client Enqueue](https://docs.dbos.dev/golang/reference/client#enqueue)
