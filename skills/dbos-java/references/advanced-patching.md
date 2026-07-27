---
title: Patch Workflows for Breaking Changes
impact: LOW
impactDescription: Lets in-flight workflows finish on old logic while new ones use new logic
tags: advanced, patching, upgrade, compatibility, deployment
---

## Patch Workflows for Breaking Changes

Changing which steps a workflow runs, or their order, breaks recovery for workflows already in flight. Patching lets
both code paths coexist in one deployment: `dbos.patch(name)` returns `true` for workflows that started after the
patch and `false` for older ones.

**Incorrect (editing a workflow in place):**

```java
@Workflow
public void workflow() {
  // Replacing foo() with baz() breaks recovery for workflows that
  // already checkpointed a "foo" step
  dbos.runStep(() -> baz(), "baz");
  dbos.runStep(() -> bar(), "bar");
}
```

**Correct (guard the new path with a patch):**

```java
// Patching must be enabled in configuration
var config = DBOSConfig.defaultsFromEnv("my-app").withEnablePatching();

@Workflow
public void workflow() {
  if (dbos.patch("use-baz")) {
    dbos.runStep(() -> baz(), "baz");   // new workflows
  } else {
    dbos.runStep(() -> foo(), "foo");   // workflows started before the patch
  }
  dbos.runStep(() -> bar(), "bar");
}
```

Retiring a patch, once every workflow that started before it has completed:

```java
// Step 1: deprecate — new workflows stop writing patch markers,
// existing ones safely skip the marker they already have
@Workflow
public void workflow() {
  if (dbos.deprecatePatch("use-baz")) {
    dbos.runStep(() -> baz(), "baz");
  }
  dbos.runStep(() -> bar(), "bar");
}

// Step 2: after all workflows with patch markers finish, remove it entirely
@Workflow
public void workflow() {
  dbos.runStep(() -> baz(), "baz");
  dbos.runStep(() -> bar(), "bar");
}
```

How it works: `patch` tries to insert a patch marker at the current point in the workflow's recorded history. If it
inserts one, or finds one already there, the workflow takes the patched path; if it finds an ordinary checkpoint
instead, the workflow predates the patch and takes the old path. `deprecatePatch` always returns `true` and skips an
existing marker.

Notes:

- Enable patching with `withEnablePatching()` (or `dbos.enable-patching=true` in Spring Boot) before using `patch`
- Removing or deprecating a patch too early, or making an unpatched breaking change, surfaces as
  `DBOSUnexpectedStepException` pointing at the offending step
- Use `dbos.listWorkflows` to confirm no active workflows remain from before the patch
- Patching keeps a single deployment; versioning runs old and new code side by side
  ([advanced-versioning.md](advanced-versioning.md))

Reference: [Patching](https://docs.dbos.dev/java/tutorials/upgrading-workflows#patching)
