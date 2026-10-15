---
title: Enqueue Workflows from External Applications
impact: HIGH
impactDescription: Enables external services to submit work to DBOS queues
tags: client, enqueue, external, queue
---

## Enqueue Workflows from External Applications

Use `dbos.Enqueue` with a client to submit workflows from outside your DBOS application. Since the Client runs externally, workflow and queue metadata must be specified explicitly by name, and the result type is given as the type parameter.

**Incorrect (trying to use RunWorkflow from external code):**

```go
// RunWorkflow requires a full DBOS context with registered workflows
dbos.RunWorkflow(ctx, processTask, "data", dbos.WithQueue(queue))
```

**Correct (using Enqueue with a client):**

```go
client, err := dbos.NewClient(context.Background(), dbos.ClientConfig{
	DatabaseURL: os.Getenv("DBOS_SYSTEM_DATABASE_URL"),
})
if err != nil {
	log.Fatal(err)
}
defer dbos.Shutdown(client, 10*time.Second)

// Basic enqueue - specify workflow and queue by name, result type as type parameter
handle, err := dbos.Enqueue[string](client, "task_queue", "processTask", "task-data")
if err != nil {
	log.Fatal(err)
}

// Wait for the result
result, err := handle.GetResult()
```

**Enqueue with options:**

```go
handle, err := dbos.Enqueue[string](client, "task_queue", "processTask", "task-data",
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
- `WithEnqueueWorkflowIDReusePolicy`: What to do if that workflow ID already exists — `WorkflowIDReusePolicyReturnExisting` (default, returns a handle to the existing workflow) or `WorkflowIDReusePolicyReject` (returns an error matching `dbos.ErrWorkflowIDInUse` without enqueueing anything). See [pattern-idempotency.md](pattern-idempotency.md)
- `WithEnqueueDeduplicationID`: Prevent duplicate enqueues
- `WithEnqueueDeduplicationPolicy`: How a colliding deduplication ID is handled — `DeduplicationPolicyReject` (default, returns a `QueueDeduplicated` error) or `DeduplicationPolicyReturnExisting` (returns a handle to the existing workflow)
- `WithEnqueuePriority`: Queue priority (lower = higher priority)
- `WithEnqueueTimeout`: Workflow timeout
- `WithEnqueueQueuePartitionKey`: Partition key for partitioned queues
- `WithEnqueueApplicationVersion`: Override application version. When unset, a client leaves the version unset and the workflow is dequeued at the owning application's latest registered version
- `WithEnqueueApplicationName`: The application that owns, dequeues, and runs the enqueued workflow (defaults to the enqueueing handle's own application). See [advanced-shared-database.md](advanced-shared-database.md)
- `WithEnqueueDelay`: Delay execution; the workflow stays in `DELAYED` status until the delay expires
- `WithEnqueueClassName`: Class/namespace name, required when enqueueing to Python, TypeScript, or Java targets
- `WithEnqueueConfigName`: Config/instance name, required when the target workflow is registered on a configured instance (Go's `WithInstance`, Python/TypeScript/Java class instances)
- `WithEnqueueAuthenticatedUser`, `WithEnqueueAssumedRole`, `WithEnqueueAuthenticatedRoles`: Attach authentication metadata (the workflow reads the user back with `dbos.GetAuthenticatedUser(ctx)`)
- `WithEnqueueTransaction(tx)`: Enqueue on a transaction you own (a `pgx.Tx`, `*sql.Tx`, or `dbos.Tx` on the DBOS system database), so the enqueue commits atomically with your own writes

The workflow name must match the registered name or custom name set with `WithWorkflowName` during registration.

**Enqueue atomically with your own writes:**

```go
tx, err := pool.Begin(ctx)
if err != nil {
	return err
}
defer tx.Rollback(ctx)
tx.Exec(ctx, "INSERT INTO orders (id) VALUES ($1)", orderID)
handle, err := dbos.Enqueue[string](client, "task_queue", "processOrder", orderID,
	dbos.WithEnqueueTransaction(tx))
if err != nil {
	return err // the transaction is aborted: roll back, do not retry on it
}
if err := tx.Commit(ctx); err != nil {
	return err // until this commits, the workflow does not exist
}
result, err := handle.GetResult() // only after the commit
```

You own the transaction: `Enqueue` neither commits nor rolls back, and the handle does not resolve until you commit. Not available inside a workflow (where an enqueue is checkpointed as a step) nor with `DeduplicationPolicyReturnExisting`.

Always call `dbos.Shutdown(client, timeout)` when done.

Reference: [DBOS Client Enqueue](https://docs.dbos.dev/golang/reference/client#enqueue)
