---
title: Schedule Workflows with the Schedule API
impact: MEDIUM
impactDescription: Run workflows exactly once per time interval with full runtime management
tags: pattern, scheduled, cron, recurring, createSchedule, applySchedules, cronTimezone, automaticBackfill
---

## Schedule Workflows with the Schedule API

Use `DBOS.createSchedule` to schedule workflows on a cron interval. Schedules are stored in the database and can be created, paused, resumed, and deleted at runtime.

**Incorrect (using the deprecated static scheduling APIs):**

```typescript
// Both APIs below are deprecated - cannot be paused, resumed, or managed at runtime

DBOS.registerScheduled(myWorkflow, { crontab: "*/30 * * * * *" });

class ScheduledExample {
  @DBOS.workflow()
  @DBOS.scheduled({ crontab: "*/30 * * * * *" })  // Also deprecated
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
  DBOS.setConfig({ name: "my-app", systemDatabaseUrl: process.env.DBOS_SYSTEM_DATABASE_URL });
  await DBOS.launch();

  // applySchedules is idempotent - safe to call on every restart
  await DBOS.applySchedules([
    { scheduleName: "my-task", workflowFn: everyFiveMinutes, schedule: "*/5 * * * *" },
  ]);
}
```

Scheduled workflow requirements:
- Must accept two arguments: `scheduledTime` (`Date`) and `context` (any serializable value)
- Not supported for workflows on instantiated objects
- `createSchedule` fails if the schedule already exists; use `applySchedules` for startup
- Scheduled workflows are automatically routed to the latest application version

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
await DBOS.registerQueue("scheduled_queue", { concurrency: 1 });

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
await DBOS.deleteSchedule("my-task");       // Remove entirely

const schedules = await DBOS.listSchedules({ status: "ACTIVE" });
const schedule = await DBOS.getSchedule("my-task");
```

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
