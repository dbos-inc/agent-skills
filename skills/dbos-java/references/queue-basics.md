---
title: Use Queues for Concurrent Workflows
impact: HIGH
impactDescription: Queues provide managed concurrency and flow control
tags: queue, concurrency, enqueue, registerQueue, flow-control
---

## Use Queues for Concurrent Workflows

Queues run many workflows with managed flow control. Register a queue with `dbos.registerQueue(name, options)`
after launch — its configuration is stored in the system database and visible to every process on that database —
then enqueue workflows with `StartWorkflowOptions.withQueue(...)`.

**Incorrect (unbounded concurrent starts):**

```java
// Starts every task immediately: no flow control, no protection for downstream services
for (String task : tasks) {
  dbos.startWorkflow(() -> proxy.processTask(task));
}
```

**Correct (enqueue with flow control):**

```java
import dev.dbos.transact.StartWorkflowOptions;
import dev.dbos.transact.workflow.QueueOptions;

dbos.launch();
dbos.registerQueue("task-queue", QueueOptions.setWorkerConcurrency(5));

@Workflow
public List<String> processAll(List<String> tasks) throws Exception {
  var options = new StartWorkflowOptions().withQueue("task-queue");

  List<WorkflowHandle<String, Exception>> handles = new ArrayList<>();
  for (String task : tasks) {
    handles.add(dbos.startWorkflow(() -> self.processTask(task), options));
  }

  List<String> results = new ArrayList<>();
  for (var handle : handles) {
    results.add(handle.getResult());
  }
  return results;
}
```

Key points:

- Enqueued workflows are dequeued in FIFO order (unless priority is enabled) by any process listening to the queue
- Enqueueing is durable: once `startWorkflow` returns, the workflow will run even if this process dies
- `QueueOptions` factories: `empty()`, `setConcurrency`, `setWorkerConcurrency`, `setRateLimit`,
  `setPriorityEnabled`, `setPartitionQueue`, `setPollingInterval`; chain more with the matching `and*` methods
- Enqueue from outside the application with `DBOSClient` ([client-enqueue.md](client-enqueue.md))
- The legacy in-memory `Queue` record with `dbos.registerQueue(Queue)` before launch is deprecated; prefer the
  database-backed form. Both accept the same settings and existing code using `new Queue("name")` still works.

To receive results as each task finishes instead of waiting in order, have each child workflow `send` a message to
the parent and `recv` them as they arrive ([comm-messages.md](comm-messages.md)).

Reference: [DBOS Queues](https://docs.dbos.dev/java/tutorials/queue-tutorial)
