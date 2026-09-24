---
title: Schedule Workflows with the Schedule API
impact: MEDIUM
impactDescription: Run workflows exactly once per time interval with full runtime management
tags: pattern, scheduled, cron, recurring, createSchedule, applySchedules, cronTimezone, automaticBackfill
---

## Schedule Workflows with the Schedule API

Use `DBOS.createSchedule` to schedule workflows on a cron interval. Schedules are stored in the database and can be created, paused, resumed, and deleted at runtime.

**Incorrect (static scheduling APIs, removed in 5.0):**

```typescript
// Removed in DBOS 5.0 - use DBOS.applySchedules / DBOS.createSchedule instead

DBOS.registerScheduled(myWorkflow, { crontab: "*/30 * * * * *" });

class ScheduledExample {
  @DBOS.workflow()
  @DBOS.scheduled({ crontab: "*/30 * * * * *" })  // Also removed
  static async scheduledWorkflow(schedTime: Date, startTime: Date) {
    // ...
  }
}
```

**Correct (using `DBOS.applySchedules` for startup schedules):**

```typescript
import { DBOS } from "@dbos-inc/dbos-sdk";

async function everyFiveMinutesFn(scheduledTime: Date, context: unknown) {
  DBOS.logger.info(`Running task scheduled for ${scheduledTime}`);
}
const everyFiveMinutes = DBOS.registerWorkflow(everyFiveMinutesFn);

async function main() {
  DBOS.setConfig({ name: "my-app", applicationVersion: "0.1.0", systemDatabaseUrl: process.env.DBOS_SYSTEM_DATABASE_URL });
  await DBOS.launch();

  // applySchedules is idempotent - safe to call on every restart
  await DBOS.applySchedules([
    { scheduleName: "my-task", workflowFn: everyFiveMinutes, schedule: "*/5 * * * *" },
  ]);
}
```

Migrating from static scheduling:
- The second argument is now the schedule's `context`, not the workflow start time
- Replace `SchedulerMode.ExactlyOncePerInterval` with `automaticBackfill: true`
- Schedules persist in the database: a schedule you stop applying keeps running until you call `DBOS.deleteSchedule`

Scheduled workflow requirements:
- Must accept two arguments: `scheduledTime` (`Date`) and `context` (any serializable value)
- Must be free functions or static class methods, not methods on `ConfiguredInstance` objects
- Schedule methods must be called **after** `DBOS.launch()` (they throw otherwise)
- `createSchedule` fails if a schedule with that name already exists; use `applySchedules` for startup
- A schedule is owned by the application (its configured `name`) that creates it: only that application's processes fire it, and its workflows run on that application's latest version
- Schedule names are globally unique across all applications sharing a system database

### `createSchedule` Parameters

`createSchedule` takes a top-level `scheduleName`/`workflowFn`/`schedule`/`context`, plus a nested `options` object for the runtime tuning fields:

```typescript
await DBOS.createSchedule({
  scheduleName: "my-task",
  workflowFn: everyFiveMinutes,
  schedule: "*/5 * * * *",
  context: "my context",
  options: {
    cronTimezone: "America/New_York",   // IANA tz; default: system local timezone
    automaticBackfill: true,            // Auto-backfill missed runs on startup
    queueName: "scheduled_queue",       // Enqueue on a declared queue
  },
});
```

`applySchedules` accepts the same fields **flattened** (no nested `options`):

```typescript
await DBOS.applySchedules([
  {
    scheduleName: "my-task",
    workflowFn: everyFiveMinutes,
    schedule: "*/5 * * * *",
    cronTimezone: "America/New_York",
    automaticBackfill: true,
    queueName: "scheduled_queue",
  },
]);
```

### Routing Scheduled Workflows to a Queue

By default, scheduled workflows run on an internal queue. Set `queueName` to enforce concurrency or rate limits:

```typescript
await DBOS.registerQueue("scheduled_queue", { globalConcurrency: 1 });

await DBOS.createSchedule({
  scheduleName: "my-task",
  workflowFn: everyFiveMinutes,
  schedule: "*/5 * * * *",
  options: { queueName: "scheduled_queue" },
});
```

### Cron Timezone

Cron expressions default to the **system's local timezone**. Set `cronTimezone` to an IANA timezone to evaluate explicitly:

```typescript
await DBOS.createSchedule({
  scheduleName: "daily-9am-ny",
  workflowFn: dailyTask,
  schedule: "0 9 * * *",
  options: { cronTimezone: "America/New_York" },
});
```

### Automatic Backfill

Set `automaticBackfill: true` so missed executions are re-run on startup or when a paused schedule resumes. Otherwise, use `DBOS.backfillSchedule` manually (see below).

### Dynamic Per-Entity Schedules

Use `createSchedule` for schedules created dynamically at runtime:

```typescript
async function customerWorkflowFn(scheduledTime: Date, customerId: string) {
  // ...
}
const customerWorkflow = DBOS.registerWorkflow(customerWorkflowFn);

async function onCustomerRegistration(customerId: string) {
  await DBOS.createSchedule({
    scheduleName: `customer-${customerId}-sync`,
    workflowFn: customerWorkflow,
    schedule: "0 * * * *",
    context: customerId,
  });
}
```

### Managing Schedules at Runtime

```typescript
await DBOS.pauseSchedule("my-task");        // Stop firing
await DBOS.resumeSchedule("my-task");       // Resume firing

const schedules = await DBOS.listSchedules({ status: "ACTIVE" });
const schedule = await DBOS.getSchedule("my-task");

// Change only some fields, preserving status and last-fired time
await DBOS.updateSchedule("my-task", { schedule: "0 * * * *", queueName: null });

// Every run is tagged with its schedule's name
const runs = await DBOS.listWorkflows({ scheduleName: "my-task" });

await DBOS.deleteSchedule("my-task");       // Remove entirely
```

`applySchedules` replaces a schedule's entire definition (omitted optional fields are cleared); `updateSchedule` changes only the fields you pass (`null` clears `cronTimezone`/`queueName`) and cannot change the workflow.

`listSchedules` and `getSchedule` return `WorkflowSchedule` objects:

```typescript
interface WorkflowSchedule {
  scheduleId: string;
  scheduleName: string;
  workflowName: string;
  workflowClassName: string;
  schedule: string;
  status: string;              // "ACTIVE" or "PAUSED"
  context: unknown;
  lastFiredAt: string | null;
  automaticBackfill: boolean;
  cronTimezone: string | null; // null = system local time
  queueName: string | null;    // null = internal queue
  applicationName?: string;    // owning application
}
```

### Manual Backfill and Trigger

Backfill missed executions (already-executed times are automatically skipped):

```typescript
await DBOS.backfillSchedule(
  "my-task",
  new Date("2025-01-01T00:00:00Z"),
  new Date("2025-01-02T00:00:00Z"),
);
```

Immediately trigger a schedule once:

```typescript
const handle = await DBOS.triggerSchedule("my-task");
```

### Crontab Format

```text
┌────────────── second (optional)
│ ┌──────────── minute
│ │ ┌────────── hour
│ │ │ ┌──────── day of month
│ │ │ │ ┌────── month
│ │ │ │ │ ┌──── day of week
* * * * * *
```

Common patterns: `* * * * *` (every minute), `0 * * * *` (hourly), `0 0 * * *` (daily), `0 0 * * 0` (weekly Sunday).

Reference: [Scheduling Workflows](https://docs.dbos.dev/typescript/tutorials/scheduled-workflows)
