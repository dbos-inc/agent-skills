---
title: Use Patching for Safe Workflow Upgrades
impact: LOW
impactDescription: Safely deploy breaking workflow changes without disrupting in-progress workflows
tags: advanced, patching, upgrade, breaking-change
---

## Use Patching for Safe Workflow Upgrades

Use `DBOS.patch()` to safely deploy breaking changes to workflow code. Breaking changes alter which steps run or their order, which can cause recovery failures.

Patching MUST be enabled in configuration; otherwise `DBOS.patch()` and `DBOS.deprecatePatch()` throw:

```typescript
DBOS.setConfig({
  name: "my-app",
  systemDatabaseUrl: process.env.DBOS_SYSTEM_DATABASE_URL,
  enablePatching: true,
});
```

If patching is enabled and `applicationVersion` is not set, DBOS uses the fixed application version `PATCHING_ENABLED` instead of a source-code hash, so workflows started before a deploy can be recovered by processes running newer code.

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

`DBOS.patch()` and `DBOS.deprecatePatch()` must be called from a workflow.

Use `DBOS.listWorkflows` to check for active old workflows before deprecating or removing patches.

**Upgrading from DBOS 4.x to 5.0:** DBOS 4.x cannot process workflows created by 5.0, and patching apps share one application version across deploys. Shut down all DBOS 4.x processes before launching DBOS 5.0 processes (see `advanced-upgrading.md`).

Reference: [Patching](https://docs.dbos.dev/typescript/tutorials/upgrading-workflows#patching)
