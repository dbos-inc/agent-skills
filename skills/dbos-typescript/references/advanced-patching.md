---
title: Use Patching for Safe Workflow Upgrades
impact: LOW
impactDescription: Safely deploy breaking workflow changes without disrupting in-progress workflows
tags: advanced, patching, upgrade, breaking-change, auto-upgrade, autoUpgrade, enterprise
---

## Use Patching for Safe Workflow Upgrades

Use `DBOS.patch()` to safely deploy breaking changes to workflow code. Breaking changes alter which steps run or their order, which can cause recovery failures.

**Incorrect (breaking change without patching):**

```typescript
// BEFORE: original workflow
async function workflowFn() {
  await foo();
  await bar();
}
const workflow = DBOS.registerWorkflow(workflowFn);

// AFTER: breaking change - recovery will fail for in-progress workflows!
async function workflowFn() {
  await baz(); // Changed step
  await bar();
}
const workflow = DBOS.registerWorkflow(workflowFn);
```

**Correct (using patch):**

```typescript
async function workflowFn() {
  if (await DBOS.patch("use-baz")) {
    await baz(); // New workflows run this
  } else {
    await foo(); // Old workflows continue with original code
  }
  await bar();
}
const workflow = DBOS.registerWorkflow(workflowFn);
```

`DBOS.patch()` returns `true` for new workflows and `false` for workflows that started before the patch.

**Deprecating patches (after all old workflows complete):**

```typescript
async function workflowFn() {
  if (await DBOS.deprecatePatch("use-baz")) { // Always returns true
    await baz();
  }
  await bar();
}
const workflow = DBOS.registerWorkflow(workflowFn);
```

**Removing patches (after all workflows using deprecatePatch complete):**

```typescript
async function workflowFn() {
  await baz();
  await bar();
}
const workflow = DBOS.registerWorkflow(workflowFn);
```

Lifecycle: `patch()` → deploy → wait for old workflows → `deprecatePatch()` → deploy → wait → remove patch entirely.

`DBOS.patch()` and `DBOS.deprecatePatch()` must be called from a workflow. They need no configuration (5.4+): `enablePatching` is deprecated and only pins `applicationVersion` to `PATCHING_ENABLED`, with a warning.

Use `DBOS.listWorkflows` to check for active old workflows before deprecating or removing patches.

### Recovering Old Workflows on Patched Code

DBOS only recovers a workflow on the application version that started it (see `advanced-versioning.md`), and the default version is a hash of your code, which changes with every patch. So patched code must be able to pick up workflows started on old code. Either:

1. **Pin the application version**: keep the same `applicationVersion` in every patched deployment:

```typescript
DBOS.setConfig({
  name: "my-app",
  systemDatabaseUrl: process.env.DBOS_SYSTEM_DATABASE_URL,
  applicationVersion: "1.0.0", // unchanged across patched deployments
});
```

2. **Auto-upgrade** (5.4+): give each deployment its own version and let DBOS move workflows left on older versions to the latest one.

### Auto-Upgrade

Auto-upgrade is a DBOS Transact Enterprise feature: it requires `@dbos-inc/dbos-enterprise` (same minor version as `@dbos-inc/dbos-sdk`) and a key whose organization is entitled to it (`conductorKey`, or `transactEnterpriseKey` in development; see `lifecycle-config.md`). **Only auto-upgrade a workflow if every breaking change to it is patched.**

```typescript
DBOS.setConfig({
  name: "my-app",
  systemDatabaseUrl: process.env.DBOS_SYSTEM_DATABASE_URL,
  autoUpgrade: true,
  autoUpgradeGraceMs: 300_000, // Optional: let old processes finish PENDING workflows first
});

// Opt a workflow with unpatched changes out (also `@DBOS.workflow({ autoUpgrade: false })`)
const workflowWithoutPatches = DBOS.registerWorkflow(workflowWithoutPatchesFn, { autoUpgrade: false });

await DBOS.launch({ conductorKey: process.env.DBOS_CONDUCTOR_KEY });
```

- `autoUpgrade: true` in a workflow's config enrolls it even when `autoUpgrade` is not set in `DBOSConfig`; `autoUpgrade: false` opts it out
- Every `autoUpgradeIntervalMs` (default 30000), one process running the latest application version moves the enrolled workflows left on older versions: `ENQUEUED` and `DELAYED` ones immediately, so latest-version processes dequeue them; `PENDING` ones, once the latest version has been the latest for `autoUpgradeGraceMs` (default 0), are re-enqueued and recovered from their last completed step. If an old process is still running one, only one execution records its outcome

**Upgrading from DBOS 4.x to 5.0:** DBOS 4.x cannot process workflows created by 5.0. If you use patching, shut down all DBOS 4.x processes before launching DBOS 5.0 processes (see `advanced-upgrading.md`).

Reference: [Patching](https://docs.dbos.dev/typescript/tutorials/upgrading-workflows#patching), [Auto-Upgrading Workflows](https://docs.dbos.dev/typescript/tutorials/upgrading-workflows#auto-upgrading-workflows)
