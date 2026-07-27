---
title: Run Workflows in the Background with startWorkflow
impact: HIGH
impactDescription: Background workflows survive crashes and let callers return immediately
tags: workflow, background, startWorkflow, handle, async
---

## Run Workflows in the Background with startWorkflow

`dbos.startWorkflow` durably starts a workflow and returns a `WorkflowHandle` without waiting for completion. Once
it returns, the workflow is guaranteed to run to completion even if the process is interrupted. This is the correct
way to run work concurrently — never spawn threads to run workflows.

**Incorrect (background work on a raw thread):**

```java
// Not durable: if the process dies, this work is lost with no record of it
new Thread(() -> proxy.backgroundTask("input")).start();
```

**Correct (durable background execution):**

```java
import dev.dbos.transact.StartWorkflowOptions;
import dev.dbos.transact.workflow.WorkflowHandle;

// Start in the background and return immediately
WorkflowHandle<String, Exception> handle =
    dbos.startWorkflow(() -> proxy.backgroundTask("input"));

String workflowId = handle.workflowId();

// Later — in this process or another one — wait for the result
String result = handle.getResult();

// Retrieve the handle again from anywhere by ID
WorkflowHandle<String, Exception> sameHandle = dbos.retrieveWorkflow(workflowId);
WorkflowStatus status = sameHandle.getStatus();
```

The lambda passed to `startWorkflow` must call exactly one workflow method on a registered proxy. Overloads accept
a `ThrowingSupplier` (workflows with a return value) or a `ThrowingRunnable` (void workflows), each with an optional
`StartWorkflowOptions`:

```java
var options = new StartWorkflowOptions()
    .withWorkflowId("order-123")            // idempotency key
    .withQueue("example-queue")             // enqueue instead of starting immediately
    .withTimeout(Duration.ofMinutes(30))    // start-to-completion timeout
    .withAttributes(Map.of("tenant", "acme")); // searchable metadata

var handle = dbos.startWorkflow(() -> proxy.backgroundTask("input"), options);
```

`WorkflowHandle<T, E>` exposes:

- `workflowId()` — the workflow's ID
- `getResult()` — block until completion, returning the result or rethrowing the workflow's exception
- `getStatus()` — the current `WorkflowStatus`

Alternatives to a handle: `dbos.getResult(workflowId)` waits for a result by ID, and
`dbos.getWorkflowStatus(workflowId)` returns an `Optional<WorkflowStatus>`. From outside the application, use
`DBOSClient` ([client-setup.md](client-setup.md)).

To run many workflows concurrently with flow control, enqueue them instead of starting them directly
([queue-basics.md](queue-basics.md)).

Reference: [Starting Workflows in the Background](https://docs.dbos.dev/java/tutorials/workflow-tutorial#starting-workflows-in-the-background)
