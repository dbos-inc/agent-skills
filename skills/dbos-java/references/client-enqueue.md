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

var client = new DBOSClient(dbUrl, dbUser, dbPassword);

var options = new DBOSClient.EnqueueOptions(
        "dataPipeline",                 // workflow name
        "com.example.DataPipelineImpl", // class name (or @WorkflowClassName value)
        "pipelineQueue")                // queue name
    .withWorkflowId(requestId)          // idempotency key
    .withPriority(10);

WorkflowHandle<String, Exception> handle =
    client.enqueueWorkflow(options, new Object[] {"task-123", "data"});

String workflowId = handle.workflowId();
String result = handle.getResult(); // optional: wait for completion
```

`EnqueueOptions` constructors take `(workflowName, queueName)` or `(workflowName, className, queueName)`; omitting
the class name makes DBOS search all registered classes for that workflow name. Options:

- `withClassName(String)` / `withInstanceName(String)` — disambiguate the target class or named instance
- `withWorkflowId(String)` — idempotency key
- `withAppVersion(String)` — pin the application version that should process the workflow
- `withTimeout(Duration)` / `withDeadline(Instant)` / `withDelay(Duration)`
- `withDeduplicationId(String)` / `withPriority(Integer)` / `withQueuePartitionKey(String)`
- `withSerialization(SerializationStrategy)` — use `PORTABLE` for cross-language arguments
- `withAttributes(Map<String, Object>)` — searchable metadata
- `withAuthenticatedUser(String)` / `withAssumedRole(String)` / `withAuthenticatedRoles(String...)`

Arguments are passed as an `Object[]` and serialized, so they must match the workflow method's parameters and be
JSON-serializable. To call a workflow implemented in Python or TypeScript, use `enqueuePortableWorkflow(options,
positionalArgs, namedArgs)` ([advanced-interops.md](advanced-interops.md)).

Workflows can also be enqueued straight from PostgreSQL — for example from a trigger — with the system database
function `dbos.enqueue_workflow(workflow_name, class_name, queue_name, positional_args)`.

Reference: [enqueueWorkflow](https://docs.dbos.dev/java/reference/client#enqueueworkflow)
