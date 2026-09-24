---
title: Inspect Workflows and Steps with listWorkflows
impact: MEDIUM
impactDescription: Enables monitoring and debugging without querying system tables directly
tags: workflow, introspection, listWorkflows, status, steps, monitoring
---

## Inspect Workflows and Steps with listWorkflows

Query workflow state through the DBOS API rather than reading system tables. `listWorkflows` filters on status,
name, time range, queue, application version, and custom attributes; `listWorkflowSteps` returns the recorded steps
of a single workflow.

**Incorrect (querying system tables directly):**

```java
// Brittle: the schema is internal and may change between releases
try (var stmt = conn.prepareStatement(
        "SELECT * FROM dbos.workflow_status WHERE status = 'ERROR'")) {
  var rs = stmt.executeQuery();
}
```

**Correct (using the introspection API):**

```java
import dev.dbos.transact.workflow.ListWorkflowsInput;
import dev.dbos.transact.workflow.WorkflowState;

// Failed workflows from the last hour, newest first
List<WorkflowStatus> failed = dbos.listWorkflows(
    new ListWorkflowsInput()
        .withStatus(WorkflowState.ERROR)
        .withStartTime(Instant.now().minus(Duration.ofHours(1)))
        .withSortDesc(true)
        .withLimit(50));

for (WorkflowStatus wf : failed) {
  System.out.printf("%s %s %s%n", wf.workflowId(), wf.workflowName(), wf.status());
}

// Steps of one workflow — functionId is the step number used by forkWorkflow
List<StepInfo> steps = dbos.listWorkflowSteps(workflowId);

// Status of a single workflow
Optional<WorkflowStatus> status = dbos.getWorkflowStatus(workflowId);
```

Useful `ListWorkflowsInput` filters (all optional, each returns a new instance):

- `withWorkflowIds(...)`, `withWorkflowIdPrefix(...)`, `withWorkflowName(...)`, `withClassName(...)`,
  `withInstanceName(...)`
- `withStatus(WorkflowState...)` — `PENDING`, `ENQUEUED`, `DELAYED`, `SUCCESS`, `ERROR`, `CANCELLED`,
  `MAX_RECOVERY_ATTEMPTS_EXCEEDED`
- `withStartTime` / `withEndTime` (creation time), `withCompletedAfter` / `withCompletedBefore`,
  `withDequeuedAfter` / `withDequeuedBefore`
- `withQueueName(...)`, `withQueuesOnly(true)` (workflows on a queue; unless you also filter by status, only
  `ENQUEUED`/`PENDING`/`DELAYED` ones — Java has no separate `listQueuedWorkflows`), `withExecutorIds(...)`,
  `withApplicationVersion(...)`, `withAuthenticatedUser(...)`
- `withApplicationName(...)` — owning applications on a shared system database; unset lists this application's
  workflows plus unclaimed ones, an empty list lists every application's workflows
  ([advanced-shared-database.md](advanced-shared-database.md))
- `withParentWorkflowId(...)`, `withHasParent(true)`, `withForkedFrom(...)`, `withWasForkedFrom(true)`
- `withScheduleName(...)` — workflows started by the named schedules; `WorkflowStatus.scheduleName()` reports the
  schedule that started a workflow (null otherwise)
- `withAttributes(Map<String, Object>)` — match workflows whose custom attributes contain these pairs
- `withLimit` / `withOffset` / `withSortDesc` for pagination and ordering
- `withLoadInput(false)` / `withLoadOutput(false)` to skip deserializing large payloads

Common queries:

```java
// Find workflows by name, filtering on multiple statuses
dbos.listWorkflows(new ListWorkflowsInput()
    .withWorkflowName("processOrder")
    .withStatus(WorkflowState.PENDING, WorkflowState.ENQUEUED));

// Find old-version workflows for blue-green deploys
dbos.listWorkflows(new ListWorkflowsInput()
    .withApplicationVersion("1.0.0")
    .withStatus(WorkflowState.PENDING, WorkflowState.ENQUEUED));

// Children of a parent, and every workflow forked from one ID
dbos.listWorkflows(new ListWorkflowsInput().withParentWorkflowId(parentId));
dbos.listWorkflows(new ListWorkflowsInput().withForkedFrom(originalId));
```

Status values:

- `ENQUEUED`: durably recorded on a queue, awaiting dequeue
- `DELAYED`: enqueued with a delay; transitions to `ENQUEUED` when the delay expires
- `PENDING`: actively executing (or about to)
- `SUCCESS` / `ERROR`: terminal
- `CANCELLED`: cancelled via `cancelWorkflow` (or timed out)
- `MAX_RECOVERY_ATTEMPTS_EXCEEDED`: exceeded retry attempts on recovery

`cancelWorkflow(id)` and `cancelWorkflows(ids)` cancel only the named workflows; pass `cancelChildren = true`
(`cancelWorkflow(id, true)`) to also recursively cancel all descendants.

`WorkflowStatus` is a record whose accessors include `workflowId`, `status`, `workflowName`, `className`,
`instanceName`, `input`, `output`, `error`, `executorId`, `appVersion`, `queueName`, `priority`,
`deduplicationId` (cleared on completion), `queuePartitionKey`, `createdAt`, `updatedAt`, `startedAt`, `completedAt`
(terminal states only), `delayUntil`, `timeout`, `deadline`, `recoveryAttempts`, `parentWorkflowId`, `forkedFrom`,
`wasForkedFrom`, `attributes`, `scheduleName`, and `applicationName`. Times are `Instant`s.

`listWorkflowSteps(workflowId, limit, offset)` paginates steps. Each `StepInfo` exposes `functionId`, `functionName`,
`output`, `error`, `childWorkflowId`, `startedAt`, and `completedAt`.

Inside a workflow or step, `DBOS.workflowId()`, `DBOS.stepId()`, `DBOS.inWorkflow()`, and `DBOS.inStep()` are static
context accessors. Attach searchable metadata when starting a workflow with
`new StartWorkflowOptions().withAttributes(Map.of("tenant", "acme"))`.

Reference: [Workflow Management Methods](https://docs.dbos.dev/java/reference/methods#workflow-management-methods)
