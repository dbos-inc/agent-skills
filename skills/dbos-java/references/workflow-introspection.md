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
- `withQueueName(...)`, `withQueuesOnly(true)`, `withExecutorIds(...)`, `withApplicationVersion(...)`
- `withParentWorkflowId(...)`, `withHasParent(true)`, `withForkedFrom(...)`, `withWasForkedFrom(true)`
- `withAttributes(Map<String, Object>)` — match workflows whose custom attributes contain these pairs
- `withLimit` / `withOffset` / `withSortDesc` for pagination and ordering
- `withLoadInput(false)` / `withLoadOutput(false)` to skip deserializing large payloads

Inside a workflow or step, `DBOS.workflowId()`, `DBOS.stepId()`, `DBOS.inWorkflow()`, and `DBOS.inStep()` are static
context accessors. Attach searchable metadata when starting a workflow with
`new StartWorkflowOptions().withAttributes(Map.of("tenant", "acme"))`.

Reference: [Workflow Management Methods](https://docs.dbos.dev/java/reference/methods#workflow-management-methods)
