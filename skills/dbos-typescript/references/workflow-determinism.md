---
title: Keep Workflows Deterministic
impact: CRITICAL
impactDescription: Non-deterministic workflows cannot recover correctly
tags: workflow, determinism, recovery, reliability
---

## Keep Workflows Deterministic

Workflow functions must be deterministic: given the same inputs and step return values, they must invoke the same steps in the same order. Non-deterministic operations must be moved to steps.

**Incorrect (non-deterministic workflow):**

```typescript
class Example {
  @DBOS.workflow()
  static async exampleWorkflow() {
    // HTTP request in workflow breaks recovery!
    const body = await fetch("https://example.com").then(r => r.text());
    await Example.processData(body);
  }
}
```

**Correct (non-determinism in step):**

```typescript
async function fetchData() {
  return await fetch("https://example.com").then(r => r.text());
}

async function exampleWorkflowFn() {
  // Step result is checkpointed for recovery
  const body = await DBOS.runStep(fetchData, { name: "fetchData" });
  await processData(body);
}
const exampleWorkflow = DBOS.registerWorkflow(exampleWorkflowFn);
```

Or using an inline arrow function:

```typescript
async function exampleWorkflowFn() {
  const body = await DBOS.runStep(
    () => fetch("https://example.com").then(r => r.text()),
    { name: "fetchData" }
  );
  await processData(body);
}
```

Non-deterministic operations that must be in steps:
- Random number generation (use `DBOS.randomUUID()` for UUIDs)
- Getting current time (use `DBOS.now()` for timestamps)
- Accessing external APIs
- Reading files
- Database queries (use transactions or steps)

Reference: [Workflow Determinism](https://docs.dbos.dev/typescript/tutorials/workflow-tutorial#determinism)
