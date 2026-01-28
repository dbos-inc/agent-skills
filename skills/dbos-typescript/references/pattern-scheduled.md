---
title: Create Scheduled Workflows
impact: MEDIUM
impactDescription: Enables recurring tasks with exactly-once-per-interval guarantees
tags: pattern, scheduled, cron, recurring
---

## Create Scheduled Workflows

Use `@DBOS.scheduled` or `DBOS.registerScheduled` to run workflows on a cron schedule. Each scheduled invocation runs exactly once per interval.

**Incorrect (manual scheduling with setInterval):**

```typescript
// Manual scheduling is not durable and misses intervals during downtime
setInterval(async () => {
  await generateReport();
}, 60000);
```

**Correct (using @DBOS.scheduled decorator):**

```typescript
import { DBOS } from "@dbos-inc/dbos-sdk";

class ScheduledTasks {
  @DBOS.workflow()
  @DBOS.scheduled({ crontab: "*/30 * * * * *" })
  static async everyThirtySeconds(scheduledTime: Date, actualTime: Date) {
    DBOS.logger.info("Running scheduled task");
  }

  @DBOS.workflow()
  @DBOS.scheduled({ crontab: "0 9 * * *" })
  static async dailyAt9AM(scheduledTime: Date, actualTime: Date) {
    await ScheduledTasks.generateReport();
  }
}
```

Using `registerScheduled` instead of decorators:

```typescript
async function dailyReportFn(scheduledTime: Date, actualTime: Date) {
  DBOS.logger.info("Generating daily report");
}
const dailyReport = DBOS.registerWorkflow(dailyReportFn);
DBOS.registerScheduled(dailyReport, { crontab: "0 9 * * *" });
```

Scheduled workflows must accept exactly two parameters: `scheduledTime` (Date) and `actualTime` (Date).

DBOS crontab supports 5 or 6 fields (optional seconds):
```text
┌────────────── second (optional)
│ ┌──────────── minute
│ │ ┌────────── hour
│ │ │ ┌──────── day of month
│ │ │ │ ┌────── month
│ │ │ │ │ ┌──── day of week
* * * * * *
```

Retroactive execution (for missed intervals):

```typescript
import { SchedulerMode } from "@dbos-inc/dbos-sdk";

class ScheduledTasks {
  @DBOS.workflow()
  @DBOS.scheduled({
    crontab: "0 21 * * 5",
    mode: SchedulerMode.ExactlyOncePerInterval,
  })
  static async fridayNightJob(scheduledTime: Date, actualTime: Date) {
    // Runs even if the app was offline during the scheduled time
  }
}
```

Scheduled workflows cannot be applied to instance methods.

Reference: [Scheduled Workflows](https://docs.dbos.dev/typescript/tutorials/scheduled-workflows)
