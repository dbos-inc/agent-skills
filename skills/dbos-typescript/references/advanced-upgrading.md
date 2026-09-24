---
title: Upgrade from DBOS TypeScript 4.x to 5.0
impact: LOW
impactDescription: Avoids code that fails on 5.x and unrecoverable workflows during the upgrade
tags: advanced, upgrade, migration, 5.0, breaking-change, removed
---

## Upgrade from DBOS TypeScript 4.x to 5.0

DBOS 5.0 removes features deprecated in 4.x. Every replacement below is also available in 4.27, so migrate code first, then upgrade. Never generate the removed APIs in new code.

### Upgrading a Running Application

DBOS 5.0 changes the storage schema for workflow inputs and outputs. 5.0 can process workflows created by 4.x, but **4.x cannot process workflows created by 5.0**:
- Don't run 4.x and 5.0 processes concurrently on the same application version. If you set `applicationVersion`, change it when you upgrade. If you use patching, shut down all 4.x processes before launching 5.0 processes.
- Upgrade `DBOSClient` users together with your DBOS processes. A 4.x client cannot read inputs or results of 5.0 workflows, and a 5.0 client requires the new schema: launch a 5.0 process (or run `npx dbos schema`) before using it.

### In-Memory Queues

**Incorrect (removed in 5.0):**

```typescript
const queue = new WorkflowQueue("example_queue", { workerConcurrency: 5 });
await DBOS.launch();
await DBOS.startWorkflow(processTask, { queueName: queue.name })(task);
```

**Correct:**

```typescript
await DBOS.launch();
await DBOS.registerQueue("example_queue", { workerConcurrency: 5 });
await DBOS.startWorkflow(processTask, { queueName: "example_queue" })(task);
```

Register every queue you previously declared in memory (workflows on an unregistered queue stay `ENQUEUED`). `listenQueues` accepts only names. Queue names starting with `_dbos_` are reserved.

Also removed: `partitionQueue: true` (use `partitionConcurrency`, `partitionWorkerConcurrency`, `partitionRateLimit`) and `priorityEnabled` (priority is always on).

```typescript
// Before: { partitionQueue: true, concurrency: 1 }
await DBOS.registerQueue("partitioned_queue", { partitionConcurrency: 1 });
```

### Static Scheduling

**Incorrect (removed in 5.0):**

```typescript
async function taskFn(scheduledTime: Date, startTime: Date) { /* ... */ }
const task = DBOS.registerWorkflow(taskFn);
DBOS.registerScheduled(task, { crontab: "*/5 * * * *" }); // also @DBOS.scheduled
```

**Correct:**

```typescript
async function taskFn(scheduledTime: Date, context: unknown) { /* ... */ }
const task = DBOS.registerWorkflow(taskFn);

await DBOS.launch();
await DBOS.applySchedules([
  { scheduleName: "my-periodic-task", workflowFn: task, schedule: "*/5 * * * *" },
]);
```

The second argument is now the schedule's `context`, not the start time. Replace `SchedulerMode.ExactlyOncePerInterval` with `automaticBackfill: true`. Schedules persist in the database: one you stop applying keeps running until you call `DBOS.deleteSchedule`.

### Calling Steps Outside Workflows

In 4.x, calling a `@DBOS.step()` method outside a workflow (or starting/enqueuing it) ran it as its own workflow. In 5.0 it runs as an ordinary function (no checkpoint, retries, or timeout), and starting or enqueuing a step throws. Call the step from a workflow instead:

```typescript
class Payments {
  @DBOS.step({ retriesAllowed: true, maxAttempts: 5 })
  static async chargeCard(orderId: string): Promise<string> {
    return await paymentApi.charge(orderId);
  }

  @DBOS.workflow()
  static async chargeCardWorkflow(orderId: string): Promise<string> {
    return await Payments.chargeCard(orderId);
  }
}

await Payments.chargeCardWorkflow(orderId); // not Payments.chargeCard(orderId)
```

Before upgrading, let any pending or enqueued 4.x step workflows (names starting with `temp_workflow-`) finish, or cancel them; 5.0 cannot recover them.

### Configuration From `dbos-config.yaml`

DBOS no longer reads `dbos-config.yaml` at launch, and `name` is required:

```typescript
DBOS.setConfig({
  name: "my-app",
  systemDatabaseUrl: process.env.DBOS_SYSTEM_DATABASE_URL,
});
await DBOS.launch();
```

The DBOS CLI and DBOS Cloud still use `dbos-config.yaml`.

### HTTP Serving and Role-Based Authorization

`@dbos-inc/koa-serve`, `DBOS.request`, `DBOS.runWithContext`, `@DBOS.requiredRole`, and `@DBOS.defaultRequiredRole` are removed. Serve HTTP with any framework (e.g., Express) and pass request data to workflows as arguments. Check roles yourself:

```typescript
class Orders {
  @DBOS.workflow()
  static async refundOrder(orderId: string): Promise<string> {
    if (!DBOS.authenticatedRoles.includes("admin")) {
      throw new Error(`User ${DBOS.authenticatedUser} is not authorized to refund orders`);
    }
    // ...
  }
}

await DBOS.withAuthedContext(user, roles, () => Orders.refundOrder(orderId));
```

`DBOS.startWorkflow` and `DBOSClient.enqueue` also accept `authenticatedUser` and `authenticatedRoles`:

```typescript
await DBOS.startWorkflow(Orders, {
  authenticatedUser: user,
  authenticatedRoles: roles,
}).refundOrder(orderId);
```

### Other Removals

- `@dbos-inc/sqs-receive`: receive with the AWS SDK and start a workflow per message, using the message ID as the workflow ID
- `@dbos-inc/pgnotifier-receiver`: enqueue from a Postgres trigger with `dbos.enqueue_workflow`
- `@dbos-inc/aws-s3-workflows`: call the AWS SDK from your own steps

Reference: [Upgrading to 5.0](https://docs.dbos.dev/typescript/upgrading)
