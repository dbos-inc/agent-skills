---
title: Schedule Workflows with the Schedule API
impact: MEDIUM
impactDescription: Run workflows exactly once per time interval with full runtime management
tags: pattern, scheduled, cron, recurring, createSchedule, applySchedules
---

## Schedule Workflows with the Schedule API

Use `DBOS.createSchedule` to schedule workflows on a cron interval. Schedules are stored in the database and can be created, paused, resumed, and deleted at runtime.

**Incorrect (using the deprecated `DBOS.registerScheduled`):**

```typescript
// Deprecated - do not use static scheduling
DBOS.registerScheduled(myWorkflow, { crontab: "*/30 * * * * *" });
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

### Backfilling and Triggering

Backfill missed executions (already-executed times are automatically skipped):

```typescript
await DBOS.backfillSchedule(
  "my-task",
  new Date("2025-01-01T00:00:00Z"),
  new Date("2025-01-02T00:00:00Z"),
);
```

Immediately trigger a schedule:

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
