---
title: Cancel, Resume, Fork, Rewind, and Delete Workflows
impact: MEDIUM
impactDescription: Recovers stuck or failed workflows without losing completed work
tags: workflow, cancel, resume, fork, rewind, management, recovery
---

## Cancel, Resume, Fork, Rewind, and Delete Workflows

DBOS manages workflow execution through the system database, so a workflow can be cancelled, resumed from its last
completed step, or forked into a new execution starting at a chosen step. Use these instead of ad-hoc retry loops
that re-run already-completed work.

**Incorrect (re-running the whole workflow after a failure):**

```java
// Restarts from scratch: every completed step runs again, duplicating side effects
try {
  proxy.processOrder(orderId);
} catch (Exception e) {
  proxy.processOrder(orderId);
}
```

**Correct (resume or fork the existing execution):**

```java
import dev.dbos.transact.workflow.ForkOptions;
import dev.dbos.transact.workflow.RewindOptions;

// Resume a cancelled or stuck workflow from its last completed step. It is re-enqueued
// on the DBOS internal queue (no flow control), so it starts as soon as it is dequeued
WorkflowHandle<String, Exception> handle = dbos.resumeWorkflow(workflowId);

// Resume onto a specific queue instead
dbos.resumeWorkflow(workflowId, "recovery-queue");

// Fork: create a NEW workflow that replays checkpoints before startStep,
// then re-executes from startStep onward with fixed code
var forked = dbos.forkWorkflow(workflowId, 2,
    new ForkOptions()
        .withApplicationVersion("2.0.0")   // run on the new code version
        .withForkedWorkflowId("retry-123"));

// Cancel: sets status to CANCELLED and preempts at the next step boundary
dbos.cancelWorkflow(workflowId);
dbos.cancelWorkflow(workflowId, true);       // also cancel descendants
dbos.cancelWorkflows(List.of("wf-1", "wf-2"));

// Delete permanently (workflow and its recorded steps)
dbos.deleteWorkflow(workflowId, true);       // also delete children

// Rewind (1.3+): re-run a FINISHED workflow in place, under the SAME ID, from step 3
dbos.rewindWorkflow(workflowId, 3);
dbos.rewindWorkflow(workflowId, 0,           // 0 re-runs the whole workflow
    new RewindOptions().withApplicationVersion("2.0.0").withQueue("recovery-queue"));
```

**Incorrect (a workflow stopping itself through the management API):**

```java
@Workflow
public void process(String id) {
  if (shouldStop()) {
    dbos.cancelWorkflow(DBOS.workflowId()); // 1.3+: throws IllegalArgumentException
  }
}
```

**Correct (return or throw from the workflow):**

```java
@Workflow
public void process(String id) {
  if (dbos.runStep(() -> shouldStop(), "shouldStop")) {
    return;
  }
}
```

Since 1.3, `cancelWorkflow(s)`, `resumeWorkflow(s)` and `deleteWorkflow(s)` called from inside a workflow, or one of
its steps, throw `IllegalArgumentException` when the calling workflow is among the targets, and change nothing. From
a workflow body the refusal is recorded as the call's step, so recovery throws it again. `DBOSClient` calls are not
affected.

Other control operations:

- `dbos.resumeWorkflows(List<String>)` / `dbos.resumeWorkflows(List<String>, String queueName)` — bulk resume
- Resume works on an `ENQUEUED` workflow too, which moves a workflow stranded on a deleted or newly partitioned
  queue; a resumed workflow keeps its application version. Workflows that ended `SUCCESS` or `ERROR` are not resumed
- `dbos.setWorkflowDelay(workflowId, Duration)` or `setWorkflowDelay(workflowId, Instant)` — hold an enqueued or
  pending workflow in `DELAYED` state until the time passes
- `dbos.updateWorkflowAttributes(workflowId, Map<String, Object>)` — replace searchable metadata; safe to call from
  inside a workflow (recorded as a step)

Notes:

- `startStep` for `forkWorkflow` is the `functionId` reported by `dbos.listWorkflowSteps(workflowId)`
- Forking creates a new workflow ID; the original is left untouched and its status reports `wasForkedFrom`, while
  the fork's `forkedFrom` names the original. List either side with `ListWorkflowsInput.withWasForkedFrom(true)` or
  `withIsFork(true)` ([workflow-introspection.md](workflow-introspection.md))
- Rewind vs. fork: rewind keeps the workflow ID (so senders, event readers and idempotency keys keep working) and
  modifies the original; fork creates a new ID. Rewind only a workflow in a terminal state (`SUCCESS`, `ERROR`,
  `CANCELLED`, `MAX_RECOVERY_ATTEMPTS_EXCEEDED`); an active one throws `IllegalStateException`, so cancel it first.
  From `startStep` on, rewind discards recorded steps and the output, rolls events back to their value before
  `startStep`, deletes messages the discarded steps consumed (and unconsumed ones), and reopens closed streams.
  `DBOS.rewindWorkflow` also deletes step-factory checkpoints past the cut, so those transactions run again.
  `DBOSClient.rewindWorkflow` can't reach the application database and leaves any it finds: a workflow that finished
  on 1.3 has none left (they're deleted when it finishes), so its transactions run again either way, while one that
  was cancelled, or finished before the 1.3 upgrade, still has them, and those steps replay their recorded results. Called from a workflow, a rewind is a step. Child
  workflows are left alone and keep their IDs, so the replay gets the existing child back; rewind a child separately
  to re-run it
- Resuming a workflow whose code changed incompatibly will fail — fork onto a new application version instead
- The same operations are available from outside the application through `DBOSClient`
  ([client-setup.md](client-setup.md))

Reference: [Workflow Management](https://docs.dbos.dev/java/tutorials/workflow-management)
