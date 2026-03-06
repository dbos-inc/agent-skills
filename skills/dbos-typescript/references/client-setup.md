---
title: Initialize DBOSClient for External Access
impact: MEDIUM
impactDescription: Enables external applications to interact with DBOS workflows
tags: client, external, setup, initialization, schedule, debounce, version
---

## Initialize DBOSClient for External Access

Use `DBOSClient` to interact with DBOS from external applications like API servers, CLI tools, or separate services. `DBOSClient` connects directly to the DBOS system database.

**Incorrect (using DBOS directly from an external app):**

```typescript
// DBOS requires full setup with launch() - too heavy for external clients
DBOS.setConfig({ name: "my-app" });
await DBOS.launch();
```

**Correct (using DBOSClient):**

```typescript
import { DBOSClient } from "@dbos-inc/dbos-sdk";

const client = await DBOSClient.create({
  systemDatabaseUrl: process.env.DBOS_SYSTEM_DATABASE_URL,
});

try {
  // Send a message to a workflow
  await client.send(workflowID, "notification", "topic");

  // Get an event from a workflow
  const event = await client.getEvent<string>(workflowID, "status");

  // Read a stream from a workflow
  for await (const value of client.readStream(workflowID, "results")) {
    console.log(value);
  }

  // Retrieve a workflow handle
  const handle = client.retrieveWorkflow<string>(workflowID);
  const result = await handle.getResult();

  // List workflows
  const workflows = await client.listWorkflows({ status: "ERROR" });

  // Workflow management
  await client.cancelWorkflow(workflowID);
  await client.resumeWorkflow(workflowID);
  await client.forkWorkflow(workflowID, stepID, { applicationVersion: "2.0.0" });

  // Wait for first of multiple workflows
  const first = await client.waitFirst(handles);
} finally {
  await client.destroy();
}
```

Constructor options:
- `systemDatabaseUrl`: Connection string to the Postgres system database (required)
- `systemDatabasePool`: Optional custom `node-postgres` connection pool
- `serializer`: Optional custom serializer (must match the DBOS application's serializer)
- `systemDatabaseSchemaName`: Optional Postgres schema name (default: `"dbos"`)

## Schedule Management

Manage workflow schedules from outside the DBOS application. Uses workflow names as strings instead of function references:

```typescript
await client.createSchedule({
  scheduleName: "my-task",
  workflowName: "myPeriodicTask",
  schedule: "*/5 * * * *",
  context: "my context",
});

const schedules = await client.listSchedules({ status: "ACTIVE" });
const schedule = await client.getSchedule("my-task");
await client.pauseSchedule("my-task");
await client.resumeSchedule("my-task");
await client.deleteSchedule("my-task");
await client.applySchedules([...]);  // Atomic batch create/update
const handles = await client.backfillSchedule("my-task", start, end);
const handle = await client.triggerSchedule("my-task");
```

## Debouncing

```typescript
import { DBOSClient, DebouncerClient } from "@dbos-inc/dbos-sdk";

const debouncer = new DebouncerClient(client, {
  workflowName: "processInput",
  debounceTimeoutMs: 120000,
});

async function onUserInput(userId: string, userInput: string) {
  await debouncer.debounce(userId, 60000, userInput);  // Wait 60s idle
}
```

## Version Management

```typescript
const versions = await client.listApplicationVersions();
const latest = await client.getLatestApplicationVersion();
await client.setLatestApplicationVersion("1.0.0");  // Rollback
```

Reference: [DBOS Client](https://docs.dbos.dev/typescript/reference/client)
