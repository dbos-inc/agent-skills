---
title: Use Patching for Safe Workflow Upgrades
impact: LOW
impactDescription: Safely deploy breaking workflow changes without disrupting in-progress workflows
tags: advanced, patching, upgrade, breaking-change
---

## Use Patching for Safe Workflow Upgrades

Use `DBOS.patch()` to safely deploy breaking changes to workflow code. Breaking changes alter which steps run or their order, which can cause recovery failures.

**Incorrect (breaking change without patching):**

```typescript
// BEFORE: original workflow
class Example {
  @DBOS.workflow()
  static async workflow() {
    await foo();
    await bar();
  }
}

// AFTER: breaking change - recovery will fail for in-progress workflows!
class Example {
  @DBOS.workflow()
  static async workflow() {
    await baz(); // Changed step
    await bar();
  }
}
```

**Correct (using patch):**

```typescript
class Example {
  @DBOS.workflow()
  static async workflow() {
    if (await DBOS.patch("use-baz")) {
      await baz(); // New workflows run this
    } else {
      await foo(); // Old workflows continue with original code
    }
    await bar();
  }
}
```

`DBOS.patch()` returns `true` for new workflows and `false` for workflows that started before the patch.

**Deprecating patches (after all old workflows complete):**

```typescript
class Example {
  @DBOS.workflow()
  static async workflow() {
    if (await DBOS.deprecatePatch("use-baz")) { // Always returns true
      await baz();
    }
    await bar();
  }
}
```

**Removing patches (after all workflows using deprecatePatch complete):**

```typescript
class Example {
  @DBOS.workflow()
  static async workflow() {
    await baz();
    await bar();
  }
}
```

Lifecycle: `patch()` → deploy → wait for old workflows → `deprecatePatch()` → deploy → wait → remove patch entirely.

Use `DBOS.listWorkflows` to check for active old workflows before deprecating or removing patches.

Reference: [Patching](https://docs.dbos.dev/typescript/tutorials/upgrading-workflows#patching)
