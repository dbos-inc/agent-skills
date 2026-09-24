---
title: Share a System Database Between Applications
impact: LOW
impactDescription: Lets multiple applications (in any language) share one system database, isolated by application name, while deliberately calling each other's workflows
tags: advanced, application-name, shared-database, ownership, rename, enqueueWorkflowWithOptions
---

## Share a System Database Between Applications

Multiple DBOS applications, potentially in different languages, can share one system database. Each application is identified by its configured `name` and owns everything it creates: workflows, steps, queues, schedules, and application versions. Applications are isolated by default but can interoperate by naming each other.

Ownership determines which application runs what:
- A workflow is dequeued, run, and recovered only by the application that owns it
- A queue is polled only by the application that registered it, even if another application enqueues on it
- A schedule is fired only by the application that created it, and its workflows are owned by that application
- Application versions are tracked per application, so one application's deploys don't change which version its peers consider latest

Queue, schedule, and version names are globally unique across the shared database; registering a name another application owns throws. Workflow IDs are also global, so ID-addressed operations (`retrieveWorkflow`, `send`, `getEvent`, `readStream`, ...) work across applications. Listing operations (`listWorkflows`, `listQueues`, `listSchedules`, `listApplicationVersions`) default to the calling application's rows.

### Calling Another Application's Workflows

**Incorrect (enqueueing a foreign workflow without naming its owner):**

```typescript
// Owned by THIS application, not order-service,
// so order-service never runs it.
await DBOS.enqueueWorkflowWithOptions({ workflowName: "process_order", queueName: "orders" }, "order-123");
```

**Correct (naming the owning application):**

```typescript
const handle = await DBOS.enqueueWorkflowWithOptions(
  {
    workflowName: "process_order",
    queueName: "orders",
    applicationName: "order-service", // the application that implements process_order
  },
  "order-123",
);
const result = await handle.getResult(); // workflow IDs are global, so waiting works
```

`DBOS.enqueueWorkflowWithOptions` enqueues by name, without a function reference, taking the same options as `DBOSClient.enqueue` (except `duplicationPolicy`). It can be called from inside a workflow (the enqueued workflow is recorded as a child). Leave `appVersion` unset so the workflow runs on the owning application's latest version. If the target is written in another language, set `serializationType: "portable"`, or use `DBOS.enqueueWorkflowWithOptionsPortable(options, positionalArgs, namedArgs?)` for targets with named arguments (e.g., Python kwargs).

To enqueue another application's workflow with a local function reference, set `enqueueOptions.applicationName` on `DBOS.startWorkflow`.

### Clients Must Name Their Application

A client with no `applicationName` sees every application's rows, but everything it creates is owned by **no** application. Unowned rows are treated as everyone's: any application may dequeue an unowned workflow, and every application fires unowned schedules and polls unowned queues. Always set `applicationName` when the database is shared:

```typescript
const client = await DBOSClient.create({
  systemDatabaseUrl: process.env.DBOS_SYSTEM_DATABASE_URL!,
  applicationName: "order-service", // act on behalf of this application
});
```

### Per-Application Options

- `applicationName` in `client.enqueue` options and `DBOS.startWorkflow`'s `enqueueOptions`: the application that owns and runs the workflow
- `registerQueue(name, { applicationName })` (client): the owning application (defaults to the client's; registering a queue owned by another application throws)
- `createSchedule({ ..., applicationName })` / `applySchedules([{ ..., applicationName }])` (client): the application that owns the schedule and runs its workflows
- `new Debouncer({ workflow, applicationName })` / `new DebouncerClient(client, { ..., applicationName })`: debounce on behalf of another application
- `setLatestApplicationVersion(version, { applicationName })`: the application to act as (promoting a version registered by a different application throws)
- Listing filters: `listWorkflows({ applicationName })`, `listQueues(applicationName)`, `listSchedules({ applicationName })` accept a name or array; rows owned by no application are always included

### Renaming an Application

Ownership is recorded under the application's name, so renaming requires transferring its rows. Stop the application first (a running application would race the rename), then:

```typescript
const counts = await client.renameApplication("old-name", "new-name", {
  // adoptUnclaimedRows: true, // also take rows owned by no application
});
// counts: { queues, schedules, versions, workflows, steps }
```

Or with the CLI: `npx dbos rename-application -s $DBOS_SYSTEM_DATABASE_URL --from old-name --to new-name` (or `dbosctl sysdb rename-application --from old-name --to new-name --db-url $DBOS_SYSTEM_DATABASE_URL`). Pass `-y` to skip the `npx dbos` confirmation prompt; `dbosctl` requires `--force` when running non-interactively. `renameApplication` is idempotent; re-running it resumes where it left off. Before adding a second application to an existing system database, adopt pre-existing unowned rows into the first one: `npx dbos rename-application -s $DBOS_SYSTEM_DATABASE_URL --to my-app --adopt-unclaimed-rows` (or `dbosctl sysdb rename-application --to my-app --adopt-unclaimed-rows`).

Reference: [Sharing a System Database](https://docs.dbos.dev/explanations/sharing-a-system-database)
