---
title: Cancel, Resume, Fork, and Delete Workflows
impact: MEDIUM
impactDescription: Recovers stuck or failed workflows without losing completed work
tags: workflow, cancel, resume, fork, management, recovery
---

## Cancel, Resume, Fork, and Delete Workflows

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
// Resume a cancelled, failed, or stuck workflow from its last completed step
WorkflowHandle<String, Exception> handle = dbos.resumeWorkflow(workflowId);

// Resume onto a queue instead of starting immediately
dbos.resumeWorkflow(workflowId, "recovery-queue");

// Fork: create a NEW workflow that replays checkpoints before startStep,
// then re-executes from startStep onward with fixed code
import dev.dbos.transact.workflow.ForkOptions;

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
```

Other control operations:

- `dbos.resumeWorkflows(List<String>)` / `dbos.resumeWorkflows(List<String>, String queueName)` — bulk resume
- `dbos.setWorkflowDelay(workflowId, Duration)` or `setWorkflowDelay(workflowId, Instant)` — hold an enqueued or
  pending workflow in `DELAYED` state until the time passes
- `dbos.updateWorkflowAttributes(workflowId, Map<String, Object>)` — replace searchable metadata; safe to call from
  inside a workflow (recorded as a step)

Notes:

- `startStep` for `forkWorkflow` is the `functionId` reported by `dbos.listWorkflowSteps(workflowId)`
- Forking creates a new workflow ID; the original is left untouched and its status reports `wasForkedFrom`
- Resuming a workflow whose code changed incompatibly will fail — fork onto a new application version instead
- The same operations are available from outside the application through `DBOSClient`
  ([client-setup.md](client-setup.md))

Reference: [Workflow Management](https://docs.dbos.dev/java/tutorials/workflow-management)
