---
title: Keep Workflows Deterministic
impact: CRITICAL
impactDescription: Non-deterministic workflows cannot recover correctly
tags: workflow, determinism, recovery, reliability
---

## Keep Workflows Deterministic

A workflow method must be deterministic: given the same inputs and the same step return values, it must invoke the
same steps in the same order. During recovery DBOS replays the workflow and reuses recorded step outputs, so any
non-deterministic operation performed directly in the workflow body can send the replay down a different path.

**Incorrect (non-determinism in the workflow body):**

```java
@Workflow
public String exampleWorkflow() {
  // A new random value on every replay — recovery may take a different branch
  if (new Random().nextInt(2) == 0) {
    return dbos.runStep(() -> stepOne(), "stepOne");
  }
  return dbos.runStep(() -> stepTwo(), "stepTwo");
}
```

**Correct (non-determinism inside a step):**

```java
@Workflow
public String exampleWorkflow() {
  // The step result is checkpointed — replay reuses the recorded value
  int choice = dbos.runStep(() -> new Random().nextInt(2), "generateChoice");
  if (choice == 0) {
    return dbos.runStep(() -> stepOne(), "stepOne");
  }
  return dbos.runStep(() -> stepTwo(), "stepTwo");
}
```

Operations that must live in steps, not in the workflow body:

- Random number generation
- Reading the current time (`Instant.now()`, `System.currentTimeMillis()`)
- HTTP requests and other calls to external services
- File system and database access
- Anything using Java's threading and concurrency APIs (`ExecutorService`, `CompletableFuture`, ...) — start
  concurrent work with `dbos.startWorkflow` and queues instead

Use `dbos.sleep(Duration)` rather than `Thread.sleep` inside workflows so the wake-up time survives restarts (see
[pattern-sleep.md](pattern-sleep.md)).

Reference: [Workflow Determinism](https://docs.dbos.dev/java/tutorials/workflow-tutorial#determinism)
