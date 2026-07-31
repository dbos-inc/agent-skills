---
title: Version Applications for Safe Upgrades
impact: LOW
impactDescription: Prevents unsafe recovery of workflows against changed code
tags: advanced, versioning, deployment, blue-green, recovery
---

## Version Applications for Safe Upgrades

Every workflow is tagged with the application version that started it, and DBOS only recovers workflows whose
version matches the running executor. Set an explicit version so deployments are deliberate rather than driven by a
source-code hash that changes with any edit.

**Incorrect (letting the version drift implicitly):**

```java
// No version configured: DBOS hashes workflow source code, so an unrelated
// edit silently produces a new version and old workflows stop recovering
var config = DBOSConfig.defaultsFromEnv("my-app");
```

**Correct (explicit version, bumped only for breaking changes):**

```java
var config = DBOSConfig.defaultsFromEnv("my-app")
    .withAppVersion("1.0.0");
```

Deployment guidance:

- New applications start at `"0.1.0"`; leave an existing application's version alone unless you are deploying a
  breaking workflow change (adding, removing, or reordering steps)
- Prefer blue-green upgrades: start processes on the new version while the old ones keep running, route new traffic
  to the new version, and retire the old processes once their workflows drain
- Check for remaining work with
  `dbos.listWorkflows(new ListWorkflowsInput().withApplicationVersion("1.0.0").withStatus(WorkflowState.PENDING))`
- The version can also be supplied per workflow with `StartWorkflowOptions.withAppVersion(...)` or
  `EnqueueOptions.withAppVersion(...)`

Version management APIs:

```java
List<VersionInfo> versions = dbos.listApplicationVersions();  // newest first
VersionInfo latest = dbos.getLatestApplicationVersion();
dbos.setLatestApplicationVersion("2.0.0");                    // promote during cutover
```

`VersionInfo` carries `versionId`, `versionName`, `versionTimestamp`, and `createdAt`. The promoted "latest"
version determines which executors may claim queued workflows that have no version assigned.

A workflow's version cannot be changed in place. To move an in-flight workflow onto new code, fork it with
`dbos.forkWorkflow(workflowId, startStep, new ForkOptions().withApplicationVersion("2.0.0"))`
([workflow-control.md](workflow-control.md)). To make a change that both versions can execute, use patching
([advanced-patching.md](advanced-patching.md)).

Reference: [Upgrading Workflow Code](https://docs.dbos.dev/java/tutorials/upgrading-workflows#versioning)
