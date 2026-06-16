---
title: List and Inspect Workflows
impact: CRITICAL
impactDescription: Enables monitoring and debugging of workflow executions
tags: workflow, list, inspect, status, monitoring
---

## List and Inspect Workflows

Use `DBOS.listWorkflows` to query workflow executions by status, name, time range, and other criteria.

**Incorrect (no monitoring of workflow state):**

```typescript
// Start workflow with no way to check on it later
await DBOS.startWorkflow(processTask)("data");
// If something goes wrong, no way to find or debug it
```

**Correct (listing and inspecting workflows):**

```typescript
const erroredWorkflows = await DBOS.listWorkflows({
  status: "ERROR",
});

for (const wf of erroredWorkflows) {
  console.log(`Workflow ${wf.workflowID}: ${wf.workflowName} - ${wf.error}`);
}
```

Common queries:

```typescript
// Find failed workflows
const failed = await DBOS.listWorkflows({ status: "ERROR", limit: 100 });

// Find workflows by name, filtering on multiple statuses
const processing = await DBOS.listWorkflows({
  workflowName: "processOrder",
  status: ["PENDING", "ENQUEUED"],
});

// Find workflows on a specific queue
const queued = await DBOS.listWorkflows({ queueName: "high_priority" });

// Only queued workflows (shortcut)
const stillQueued = await DBOS.listQueuedWorkflows({ queueName: "task_queue" });

// Find old-version workflows for blue-green deploys
const old = await DBOS.listWorkflows({
  applicationVersion: "1.0.0",
  status: ["PENDING", "ENQUEUED"],
});

// Find children of a parent workflow
const children = await DBOS.listWorkflows({ parentWorkflowID: parentId });

// Find every workflow forked from one ID
const forks = await DBOS.listWorkflows({ forkedFrom: originalId });

// Sort newest-first, paginate
const page = await DBOS.listWorkflows({ limit: 50, offset: 100, sortDesc: true });
```

### Filter Fields (`GetWorkflowsInput`)

- **workflowIDs**: Specific IDs to fetch
- **workflowName**: Workflow function name (or array)
- **status**: Single status or array (see status values below)
- **startTime** / **endTime**: RFC 3339 timestamps
- **completedAfter** / **completedBefore**: RFC 3339 timestamps; completed at-or-after / at-or-before that timestamp
- **dequeuedAfter** / **dequeuedBefore**: RFC 3339 timestamps; dequeued at-or-after / at-or-before that timestamp
- **applicationVersion**: Version(s) the workflow ran on
- **executorId**: Executor process ID(s)
- **workflow_id_prefix**: Match workflows whose IDs start with this
- **authenticatedUser**: User(s) who ran the workflow
- **queueName**: Queue name(s)
- **queuesOnly**: If `true`, only currently-enqueued workflows (same as `listQueuedWorkflows`)
- **forkedFrom**: Source workflow ID(s) for forks
- **wasForkedFrom**: `true` for workflows that have been forked from, `false` for those that haven't
- **parentWorkflowID**: Parent workflow ID(s)
- **hasParent**: `true` for child workflows only, `false` for top-level only
- **limit** / **offset**: Pagination
- **sortDesc**: Sort by creation time descending (default ascending)
- **loadInput** / **loadOutput**: Set to `false` to skip deserializing for performance

### Status Values

`ENQUEUED`, `DELAYED`, `PENDING`, `SUCCESS`, `ERROR`, `CANCELLED`, `MAX_RECOVERY_ATTEMPTS_EXCEEDED`

- `ENQUEUED`: durably recorded on a queue, awaiting dequeue
- `DELAYED`: enqueued with `delaySeconds`; transitions to `ENQUEUED` when the delay expires
- `PENDING`: actively executing (or about to)
- `SUCCESS` / `ERROR`: terminal
- `CANCELLED`: cancelled via `cancelWorkflow` (or timed out)
- `MAX_RECOVERY_ATTEMPTS_EXCEEDED`: exceeded retry attempts on recovery

`cancelWorkflow(workflowID, options?: { cancelChildren?: boolean })` and `cancelWorkflows(workflowIDs, options?: { cancelChildren?: boolean })` cancel workflows. Child workflows are not cancelled by default; pass `{ cancelChildren: true }` to also recursively cancel all child workflows.

### `WorkflowStatus` Fields

```typescript
interface WorkflowStatus {
  workflowID: string;
  status: string;
  workflowName: string;
  workflowClassName: string;
  workflowConfigName?: string;
  queueName?: string;

  authenticatedUser?: string;
  assumedRole?: string;
  authenticatedRoles?: string[];

  input?: unknown[];
  output?: unknown;
  error?: unknown;

  executorId?: string;
  applicationVersion?: string;

  createdAt: number;           // Unix epoch ms
  updatedAt?: number;
  completedAt?: number;        // The time the workflow completed (SUCCESS, ERROR, or CANCELLED), as a Unix epoch timestamp in milliseconds. Undefined if not completed.

  timeoutMS?: number;
  deadlineEpochMS?: number;    // Computed deadline from start time + timeoutMS
  deduplicationID?: string;    // Cleared on completion
  priority: number;            // 0 = highest (unset)
  queuePartitionKey?: string;
  dequeuedAt?: number;

  forkedFrom?: string;
  wasForkedFrom?: boolean;
  parentWorkflowID?: string;
}
```

### Listing Steps

```typescript
const steps = await DBOS.listWorkflowSteps(workflowID, { limit: 100, offset: 0 });
if (steps) {
  for (const step of steps) {
    console.log(`Step ${step.functionID}: ${step.name}`);
    if (step.error) console.log(`  Error: ${step.error}`);
    if (step.childWorkflowID) console.log(`  Child: ${step.childWorkflowID}`);
    if (step.startedAtEpochMs && step.completedAtEpochMs) {
      console.log(`  Duration: ${step.completedAtEpochMs - step.startedAtEpochMs}ms`);
    }
  }
}
```

Each `StepInfo` exposes: `functionID`, `name`, `output`, `error`, `childWorkflowID`, `startedAtEpochMs`, `completedAtEpochMs`. Returns `undefined` if the workflow is not found.

To optimize performance, set `loadInput: false` and `loadOutput: false` on `listWorkflows` when you don't need workflow inputs or outputs.

Reference: [Workflow Management](https://docs.dbos.dev/typescript/tutorials/workflow-management)
