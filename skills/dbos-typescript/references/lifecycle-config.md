---
title: Configure and Launch DBOS Properly
impact: CRITICAL
impactDescription: Application won't function without proper setup
tags: configuration, launch, setup, initialization
---

## Configure and Launch DBOS Properly

Every DBOS application must configure and launch DBOS before running any workflows. All workflows and steps must be registered before calling `DBOS.launch()`.

**Incorrect (missing configuration or launch):**

```typescript
import { DBOS } from "@dbos-inc/dbos-sdk";

// No configuration or launch!
async function myWorkflowFn() {
  // This will fail - DBOS is not launched
}
const myWorkflow = DBOS.registerWorkflow(myWorkflowFn);
await myWorkflow();
```

**Correct (configure and launch in main):**

```typescript
import { DBOS } from "@dbos-inc/dbos-sdk";

async function myWorkflowFn() {
  // workflow logic
}
const myWorkflow = DBOS.registerWorkflow(myWorkflowFn);

async function main() {
  DBOS.setConfig({
    name: "my-app",
    systemDatabaseUrl: process.env.DBOS_SYSTEM_DATABASE_URL,
  });
  await DBOS.launch();
  await myWorkflow();
}

main().catch(console.log);
```

For scheduled-only applications, create schedules after launch:

```typescript
async function main() {
  DBOS.setConfig({ name: "my-app", systemDatabaseUrl: process.env.DBOS_SYSTEM_DATABASE_URL });
  await DBOS.launch();
  await DBOS.applySchedules([
    { scheduleName: "my-task", workflowFn: scheduledTask, schedule: "* * * * *" },
  ]);
}
```

## DBOSConfig Reference

All fields except `name` are optional:

| Field | Description | Default |
|-------|-------------|---------|
| **name** | Application name | (required) |
| **systemDatabaseUrl** | Postgres connection string for system DB | `postgresql://postgres:dbos@localhost:5432/[name]_dbos_sys` |
| **applicationVersion** | Version tag for versioning strategy | Auto-computed hash |
| **executorID** | Unique process ID for distributed environments | Auto-set by Conductor |
| **systemDatabasePoolSize** | System DB connection pool size | `10` |
| **systemDatabaseSchemaName** | Postgres schema for DBOS system tables | `"dbos"` |
| **systemDatabasePool** | Custom `node-postgres` pool (skips pool creation) | `undefined` |
| **enableOTLP** | Enable OpenTelemetry tracing and export | `false` |
| **logLevel** | DBOS logger severity | `"info"` |
| **otlpTracesEndpoints** | OTLP trace receiver URLs | `undefined` |
| **otlpLogsEndpoints** | OTLP log receiver URLs | `undefined` |
| **runAdminServer** | Run HTTP admin server | `true` |
| **adminPort** | Admin server port | `3001` |
| **listenQueues** | Only listen to these queues | All declared queues |
| **schedulerPollingIntervalMs** | Scheduler polling interval for new schedules (ms) | `30000` |
| **serializer** | Custom serializer for system database | Default (JSON/SuperJSON) |

Reference: [DBOS Configuration](https://docs.dbos.dev/typescript/reference/configuration)
