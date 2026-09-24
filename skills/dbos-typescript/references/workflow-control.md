---
title: Cancel, Resume, Fork, and Rewind Workflows
impact: CRITICAL
impactDescription: Enables operational control over long-running workflows
tags: workflow, cancel, resume, fork, rewind, management
---

## Cancel, Resume, Fork, and Rewind Workflows

DBOS provides methods to cancel, resume, fork, and rewind workflows for operational control.

**Incorrect (no way to handle stuck or failed workflows):**

```typescript
// Workflow is stuck or failed - no recovery mechanism
const handle = await DBOS.startWorkflow(processTask)("data");
// If the workflow fails, there's no way to retry or recover
```

**Correct (using cancel, resume, and fork):**

```typescript
// Cancel a workflow - stops at its next step
await DBOS.cancelWorkflow(workflowID);

// Resume from the last completed step
const handle = await DBOS.resumeWorkflow<string>(workflowID);
const result = await handle.getResult();
```

Cancellation sets the workflow status to `CANCELLED`, removes it from its queue, and preempts execution at the beginning of the next step. A step already running is not interrupted, but it can stop early by passing `DBOS.stepStatus.cancelSignal` to APIs like `fetch` (see `step-timeouts.md`). Child workflows are not cancelled by default; pass `{ cancelChildren: true }` to also recursively cancel all child workflows:

```typescript
// cancelWorkflow(workflowID, options?: { cancelChildren?: boolean }) — defaults to false
await DBOS.cancelWorkflow(workflowID, { cancelChildren: true });
```

The same `options?: { cancelChildren?: boolean }` is available on `cancelWorkflows(workflowIDs, options?)`.

Resume restarts a workflow from its last completed step. Use this for workflows that are cancelled or have exceeded their maximum recovery attempts. You can also use this to start an enqueued workflow immediately, bypassing its queue.

Fork a workflow from a specific step:

```typescript
// List steps to find the right step ID
const steps = await DBOS.listWorkflowSteps(workflowID);
// steps[i].functionID is the step's ID

// Fork from a specific step
const forkHandle = await DBOS.forkWorkflow<string>(
  workflowID,
  startStep,
  {
    newWorkflowID: "new-wf-id",
    applicationVersion: "2.0.0", // Defaults to the original workflow's version
    timeoutMS: 60000,
    queueName: "recovery_queue",      // Optional: enqueue instead of starting immediately
    queuePartitionKey: "customer-42", // Required if queueName is a partitioned queue
    replacementChildren: { "old-child-id": "forked-child-id" }, // Substitute forked children
  }
);
const forkResult = await forkHandle.getResult();
```

Forking creates a new workflow with a new ID, copying the original workflow's inputs and step outputs up to the selected step. Useful for recovering from downstream service outages or patching workflows that failed due to a bug. `replacementChildren` maps original child workflow IDs to replacement IDs, for forking a parent whose children were also forked.

### Rewinding a Workflow

`DBOS.rewindWorkflow` re-executes a workflow from a step **in place**, keeping its workflow ID (fork creates a copy with a new ID). Use it when other code refers to the workflow by ID (e.g., an idempotency key derived from an order ID): senders, event/stream readers, and child workflow IDs keep working.

```typescript
// Re-execute from step 3, keeping the workflow ID
const handle = await DBOS.rewindWorkflow<string>(workflowID, {
  startStep: 3,                // Default 0: re-execute the whole workflow
  applicationVersion: "2.0.0", // Optional: rewind onto fixed code
  // queueName / queuePartitionKey: optional; defaults to an internal queue
});
const result = await handle.getResult();
```

Rewind rules:
- Only a workflow in a terminal state (`SUCCESS`, `ERROR`, `CANCELLED`, `MAX_RECOVERY_ATTEMPTS_EXCEEDED`) can be rewound; cancel a running workflow first
- Steps `>= startStep` are discarded and re-executed; earlier steps replay their recorded outputs
- Clears the output/error, restores events set at or after `startStep` to their earlier values, and deletes messages consumed at or after `startStep` (and unconsumed messages)
- Streams are append-only: new values are appended, and streams closed at or after `startStep` are reopened
- Child workflows are not modified; delete or rewind them separately
- `DBOS.rewindWorkflow` deletes data source transaction checkpoints for the rewound steps. `client.rewindWorkflow` (same options) does **not**, so rewound transactions are not re-executed; use `DBOS.rewindWorkflow` for workflows that use data sources

Reference: [Workflow Management](https://docs.dbos.dev/typescript/tutorials/workflow-management)
