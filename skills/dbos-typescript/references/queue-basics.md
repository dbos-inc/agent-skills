---
title: Use Queues for Concurrent Workflows
impact: HIGH
impactDescription: Queues provide managed concurrency and flow control
tags: queue, concurrency, enqueue, workflow
---

## Use Queues for Concurrent Workflows

Queues run many workflows concurrently with managed flow control. Use them when you need to control how many workflows run at once.

**Incorrect (uncontrolled concurrency):**

```typescript
async function processTaskFn(task: string) {
  // ...
}
const processTask = DBOS.registerWorkflow(processTaskFn);

// Starting many workflows without control - could overwhelm resources
for (const task of tasks) {
  await DBOS.startWorkflow(processTask)(task);
}
```

**Correct (using a queue):**

```typescript
import { DBOS, WorkflowQueue } from "@dbos-inc/dbos-sdk";

const queue = new WorkflowQueue("task_queue");

async function processTaskFn(task: string) {
  // ...
}
const processTask = DBOS.registerWorkflow(processTaskFn);

async function processAllTasksFn(tasks: string[]) {
  const handles = [];
  for (const task of tasks) {
    // Enqueue by passing queueName to startWorkflow
    const handle = await DBOS.startWorkflow(processTask, {
      queueName: queue.name,
    })(task);
    handles.push(handle);
  }
  // Wait for all tasks
  const results = [];
  for (const h of handles) {
    results.push(await h.getResult());
  }
  return results;
}
const processAllTasks = DBOS.registerWorkflow(processAllTasksFn);
```

With decorators:

```typescript
const queue = new WorkflowQueue("task_queue");

class Tasks {
  @DBOS.workflow()
  static async processTask(task: string) {
    // ...
  }

  @DBOS.workflow()
  static async processAllTasks(tasks: string[]) {
    const handles = [];
    for (const task of tasks) {
      handles.push(
        await DBOS.startWorkflow(Tasks, { queueName: queue.name }).processTask(task)
      );
    }
    const results = [];
    for (const h of handles) {
      results.push(await h.getResult());
    }
    return results;
  }
}
```

Queues process workflows in FIFO order. All queues should be created before `DBOS.launch()`.

Reference: [DBOS Queues](https://docs.dbos.dev/typescript/tutorials/queue-tutorial)
