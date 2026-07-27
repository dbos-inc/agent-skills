---
title: Follow Workflow and Step Constraints
impact: CRITICAL
impactDescription: Violating constraints breaks recovery and durability guarantees
tags: workflow, step, constraints, rules, best-practices
---

## Follow Workflow and Step Constraints

DBOS operations that create their own checkpoints — starting workflows, `send`, `recv`, `setEvent`, `getEvent`,
`sleep` — belong in workflow bodies, not in steps. Steps should do external work and return a serializable result.

**Incorrect (starting a workflow and receiving messages from a step):**

```java
@Workflow
public void myWorkflow(String input) {
  dbos.runStep(() -> {
    // Workflow operations inside a step break the checkpoint sequence
    dbos.startWorkflow(() -> self.childWorkflow(input));
    dbos.recv("topic", Duration.ofSeconds(30));
  }, "badStep");
}
```

**Correct (workflow operations from the workflow, external work from steps):**

```java
@Workflow
public String myWorkflow(String input) throws Exception {
  // External work goes in a step
  String data = dbos.runStep(() -> fetchData(input), "fetchData");

  // Workflow operations stay in the workflow body
  var handle = dbos.startWorkflow(() -> self.childWorkflow(data));
  String childResult = handle.getResult();

  dbos.setEvent("status", "child-complete");
  Optional<String> signal = dbos.recv("approval", Duration.ofMinutes(10));

  return childResult;
}
```

Additional constraints:

- Register every workflow class before `launch()`; register database-backed queues after `launch()`
- Do not mutate static or shared state from workflows or steps — read-only access to instance fields is fine, and
  per-instance configuration should use named instances ([workflow-instances.md](workflow-instances.md))
- Workflow arguments and workflow/step return values must be serializable by Jackson; add Jackson annotations where
  needed (see [advanced-serialization.md](advanced-serialization.md))
- A step may call another step, but the inner call becomes part of the outer step's execution rather than a separate
  checkpoint
- `setEvent` and `recv` can only be called from inside a workflow; `send`, `getEvent`, and `readStream` may also be
  called from outside
- Changing which steps a workflow runs, or their order, is a breaking change for in-flight workflows — use patching
  or versioning ([advanced-patching.md](advanced-patching.md), [advanced-versioning.md](advanced-versioning.md))

Reference: [Workflow Tutorial](https://docs.dbos.dev/java/tutorials/workflow-tutorial)
