---
title: Use Versioning for Blue-Green Deployments
impact: LOW
impactDescription: Enables safe deployment of new code versions alongside old ones
tags: advanced, versioning, blue-green, deployment
---

## Use Versioning for Blue-Green Deployments

Set `applicationVersion` in configuration to tag workflows with a version. DBOS only recovers workflows matching the current application version, preventing code mismatches during recovery.

**Incorrect (deploying new code that breaks in-progress workflows):**

```typescript
DBOS.setConfig({
  name: "my-app",
  systemDatabaseUrl: process.env.DBOS_SYSTEM_DATABASE_URL,
  // No version set - all workflows recovered regardless of code version
});
```

**Correct (versioned deployment):**

```typescript
DBOS.setConfig({
  name: "my-app",
  systemDatabaseUrl: process.env.DBOS_SYSTEM_DATABASE_URL,
  applicationVersion: "2.0.0",
});
```

By default, the application version is automatically computed from a hash of workflow source code. Set it explicitly for more control.

### Directing Enqueued Workflows to Latest Version

Use `DBOS.getLatestApplicationVersion` to route enqueued work to the latest version:

```typescript
const latestVersion = await DBOS.getLatestApplicationVersion();
const handle = await DBOS.startWorkflow(myWorkflow, {
  queueName: "my_queue",
  enqueueOptions: { applicationVersion: latestVersion.versionName },
})(arg1, arg2);
```

Scheduled workflows are automatically enqueued to the latest version.

### Checking and Retiring Old Versions

```typescript
const active = await DBOS.listWorkflows({
  applicationVersion: "1.0.0",
  status: ["ENQUEUED", "PENDING"],
});
if (active.length === 0) {
  console.log("Safe to retire version 1.0.0");
}
```

### Version Management APIs

```typescript
// List all registered versions (newest first)
const versions = await DBOS.listApplicationVersions();

// Get the latest version
const latest = await DBOS.getLatestApplicationVersion();

// Roll back: promote a previous version to latest
await DBOS.setLatestApplicationVersion("1.0.0");
```

### Forking Workflows to a New Version

```typescript
const handle = await DBOS.forkWorkflow<string>(
  workflowID,
  failedStepID,
  { applicationVersion: "2.0.0" }
);
```

Reference: [Versioning](https://docs.dbos.dev/typescript/tutorials/upgrading-workflows#versioning)
