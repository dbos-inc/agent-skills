---
title: Share a System Database Between Applications
impact: LOW
impactDescription: Lets multiple applications (in any language) share one system database, isolated by application name, while deliberately calling each other's workflows
tags: advanced, application-name, shared-database, ownership, rename
---

## Share a System Database Between Applications

Multiple DBOS applications, potentially in different languages, can share a single system database (Java SDK
1.1+). Each application is identified by the name passed to `DBOSConfig.defaults(appName)` /
`defaultsFromEnv(appName)` and owns everything it creates: workflows, steps, queues, schedules, and application
versions. A row with no owner (`NULL` application name — written before sharing existed, or by an unnamed client)
is unclaimed and belongs to every application.

Ownership determines which application runs what:

- A workflow is dequeued, run, and recovered only by the application that owns it (or by any, if unclaimed); the
  first dequeue claims an unclaimed workflow
- A queue is polled only by the application that registered it, even if other applications enqueue onto it
- A schedule is fired only by the application that owns it, and its workflows are owned by that application
- Application versions are tracked per application, so one application's deployments do not change which version
  its peers consider latest

Queue, schedule, and version names stay globally unique across the shared database: registering one another
application owns throws `DBOSApplicationNameConflictException` (`kind()`, `name()`, `owner()`). Workflow IDs are
global too, so ID-addressed calls (`retrieveWorkflow`, `getResult`, `send`, `getEvent`, `readStream`, cancel, resume,
fork, ...) work across applications. Listings (`listWorkflows`, `listQueues`, `listSchedules`, versions) and the
dequeue are scoped to the caller's application plus unclaimed rows. The deduplication index is global: a
deduplication ID held by a peer's workflow on the same queue blocks yours.

### Calling another application's workflows

**Incorrect (enqueueing a foreign workflow without naming its owner):**

```java
// This application owns the workflow but has no processOrder registered,
// and does not poll order-queue — the workflow is never dequeued
dbos.enqueueWorkflow(
    new DBOSClient.EnqueueOptions("processOrder", "OrderService", "order-queue"),
    new Object[] {"order-123"});
```

**Correct (naming the owning application):**

```java
var options = new DBOSClient.EnqueueOptions("processOrder", "OrderService", "order-queue")
    .withApplicationName("order-service"); // owns, dequeues, and runs it

WorkflowHandle<String, Exception> handle =
    dbos.enqueueWorkflow(options, new Object[] {"order-123"});
String result = handle.getResult(); // workflow IDs are global, so waiting works
```

`DBOS.enqueueWorkflow(EnqueueOptions, Object[])` and `DBOS.enqueuePortableWorkflow(options, positionalArgs,
namedArgs)` take the same `EnqueueOptions` as `DBOSClient` and write the same row, without a reference to the
workflow's function. Unlike `startWorkflow`, the workflow and queue are not checked against local registries. Called
inside a workflow, the enqueue is recorded as a child, so a replay returns the original handle; called from a step,
it throws `IllegalStateException`. Leave `withAppVersion` unset: an unversioned workflow is dequeued only by an
executor running the owning application's latest version. For a target in another language use
`enqueuePortableWorkflow` ([advanced-interops.md](advanced-interops.md)).

### Clients must name their application

An unnamed `DBOSClient` sees every application's rows but owns nothing: what it enqueues, registers, or schedules is
unclaimed, so any application may dequeue those workflows, and every application polls those queues and fires those
schedules. Name the client when the database is shared:

```java
var client = new DBOSClient(url, user, password, null, null, false, "order-service");
client.applicationName(); // "order-service"
```

### Per-application options

- `EnqueueOptions.withApplicationName(name)` — enqueue a workflow owned (and run) by another application
- `DBOSClient.registerQueue(name, options, onConflict, applicationName)` — register a queue on behalf of an
  application
- `WorkflowSchedule.withApplicationName(name)` — the application that owns a schedule and runs its workflows
- `DBOSClient.setLatestApplicationVersion(versionName, applicationName)` — promote a version on behalf of an
  application; `VersionInfo.applicationName()` reports the owner
- Listing filters, each covering the named applications plus unclaimed rows (`null` = own application, an empty list =
  every application): `dbos.listQueues(List<String>)`, the 4-argument `listSchedules(status, workflowName, namePrefix,
  applicationName)`, `ListWorkflowsInput.withApplicationName(...)`
- `DBOSClient.findDeduplicationHolder(queueName, deduplicationId)` — the active workflow holding a deduplication ID,
  with its owning application (it may be a peer's)

`WorkflowStatus.applicationName()` reports a workflow's owner.

### Names and versions

- Application names are only enforced where Conductor needs them: 3–256 characters of lowercase letters, digits,
  `-` and `_`. Other names log a warning, but fail launch when a Conductor key is set or on DBOS Cloud (which takes
  the name from `DBOS_APP_NAME`)
- A computed application version (no `withAppVersion`) hashes the application name, so two applications built from
  one jar do not share a version row — and upgrading to 1.1 changes existing computed versions
- An explicit version string is a globally unique name too: if two applications both set `withAppVersion("0.1.0")`,
  the second fails launch with `DBOSApplicationNameConflictException`. Give each application its own version strings,
  for example prefixed with the application name (`"billing-0.1.0"`). Enabling patching without `withAppVersion`
  sets the version to `PATCHING_ENABLED`, which collides the same way — set an explicit version alongside patching

### Renaming an application

Ownership is keyed by name, so a renamed application no longer sees its old rows. Stop the application first (a
running one keeps dequeuing under its old name), then re-own them:

```java
ApplicationRowCounts moved = client.renameApplication("old-name", "new-name");
// or: renameApplication(oldName, newName, batchSize, adoptUnclaimedRows)
// moved.queues(), schedules(), versions(), workflows(), steps()
```

Or with the CLI: `dbosctl sysdb rename-application --from old-name --to new-name`. Re-running resumes an interrupted
rename. Before adding a second application to an existing database, adopt the pre-existing unclaimed rows into the
first one: `dbosctl sysdb rename-application --to my-app --adopt-unclaimed-rows`.

Reference: [Sharing a System Database](https://docs.dbos.dev/explanations/sharing-a-system-database)
