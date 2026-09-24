---
title: Enqueue Workflows from External Applications
impact: MEDIUM
impactDescription: Lets other services trigger durable workflows without hosting them
tags: client, enqueue, external, EnqueueOptions, integration
---

## Enqueue Workflows from External Applications

`DBOSClient.enqueueWorkflow` submits a workflow to a queue by name, so an API server can hand work to a separate
processing service without linking against its code. The client is outside the application, so workflow name, class
name, and queue must be given explicitly.

**Incorrect (an ad-hoc job table):**

```java
// Custom polling, retry, and status plumbing — exactly what DBOS queues provide
jdbc.update("INSERT INTO job_queue(payload, status) VALUES (?, 'pending')", payload);
```

**Correct (enqueue through DBOSClient):**

```java
import dev.dbos.transact.DBOSClient;
import dev.dbos.transact.EnqueueOptions;
import dev.dbos.transact.workflow.QueueName;
import dev.dbos.transact.workflow.QueueOptions;

try (var client = new DBOSClient(dbUrl, dbUser, dbPassword)) {
  // Optionally register the queue from the client (persists to the system database)
  client.registerQueue("pipelineQueue", QueueOptions.setConcurrency(10));

  var options = new EnqueueOptions(
          "dataPipeline",                  // workflow name
          "com.example.DataPipelineImpl",  // class name (or @WorkflowClassName value)
          QueueName.of("pipelineQueue"))   // queue
      .withWorkflowId(requestId)           // idempotency key
      .withPriority(10);

  WorkflowHandle<String, Exception> handle =
      client.enqueueWorkflow(options, new Object[] {"task-123", "data"});

  String workflowId = handle.workflowId();
  String result = handle.getResult(); // optional: wait for completion
}
```

The queue does not need to exist when `enqueueWorkflow` is called. If no queue with the given name has been
registered, the workflow is still durably recorded as `ENQUEUED` and starts running once the queue is registered and
a worker becomes available.

`EnqueueOptions` is the top-level `dev.dbos.transact.EnqueueOptions`, shared by `DBOSClient.enqueueWorkflow` and
`dbos.enqueueWorkflow`. The constructors fix what to run and where, with the queue as a `QueueName`:
`(workflowName, queue)`, `(workflowName, className, queue)`, and `(workflowName, className, instanceName, queue)`.
There is no `withClassName` or `withInstanceName`. Without a class name, DBOS searches all registered classes for the
workflow name. The nested `DBOSClient.EnqueueOptions`, and the client overloads that take it, are deprecated for
removal since 1.1 — do not use them. Options:

- `withWorkflowId(String)` — idempotency key
- `withAppVersion(String)` — pin the application version that should process the workflow; left unset, the
  owning application's latest version dequeues it
- `withApplicationName(String)` — the application that owns and runs the workflow (default: the client's own, or
  unclaimed for an unnamed client) ([advanced-shared-database.md](advanced-shared-database.md))
- `withTimeout(Duration | long, TimeUnit | Timeout)` / `withNoTimeout()` — inside a workflow (`dbos.enqueueWorkflow`)
  an unset timeout inherits the caller's and `withNoTimeout()` declines it; from a client, unset means none. Only an
  explicit timeout conflicts with `withDeadline`
- `withDeadline(Instant)` / `withDelay(Duration)`
- `withDeduplicationId(String)` / `withPriority(Integer)` / `withQueuePartitionKey(String)`
- `withSerialization(SerializationStrategy)` — use `PORTABLE` for cross-language arguments
- `withAttributes(Map<String, Object>)` — searchable metadata
- `withAuthenticatedUser(String)` / `withAssumedRole(String)` / `withAuthenticatedRoles(String...)` /
  `withAuthentication(user, roles...)`

Arguments are passed as an `Object[]` and serialized, so they must match the workflow method's parameters and be
JSON-serializable. To call a workflow implemented in Python or TypeScript, set
`withSerialization(SerializationStrategy.PORTABLE)` and, for named arguments, use `enqueueWorkflow(options,
positionalArgs, namedArgs)`; named arguments without `PORTABLE` throw `IllegalArgumentException`
([advanced-interops.md](advanced-interops.md)).

Inside a DBOS application, `dbos.enqueueWorkflow(options, args)` and `dbos.enqueueWorkflow(options, positionalArgs,
namedArgs)` take the same `EnqueueOptions` to enqueue a workflow the application has no reference to.

Not available in Java: a `return-existing` deduplication policy (a colliding deduplication ID always throws
`DBOSQueueDuplicatedException`), a per-enqueue maximum recovery attempts, and enqueueing inside a caller-owned
database transaction.

Always `close()` the client when done.

Workflows can also be enqueued straight from PostgreSQL — for example from a trigger — with the system database
function `dbos.enqueue_workflow(workflow_name, class_name, queue_name, positional_args)`.

Reference: [enqueueWorkflow](https://docs.dbos.dev/java/reference/client#enqueueworkflow)
