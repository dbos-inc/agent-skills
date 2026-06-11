---
title: Aggregate Workflow and Step Counts for Analytics
impact: MEDIUM
impactDescription: Enables low-cost analytics over workflow and step status without scanning the workflow table
tags: workflow, step, aggregates, analytics, observability
---

## Aggregate Workflow and Step Counts for Analytics

`dbos.GetWorkflowAggregates` returns grouped counts of workflows. Use it for dashboards and queue-health checks instead of listing every workflow and counting in application code.

**Incorrect (N+1, paginates the whole workflow table):**

```go
workflows, _ := dbos.ListWorkflows(ctx,
    dbos.WithStartTime(time.Now().Add(-24*time.Hour)))
counts := map[string]int{}
for _, w := range workflows {
    counts[string(w.Status)]++
}
```

**Correct (single aggregate query):**

```go
rows, err := dbos.GetWorkflowAggregates(ctx, dbos.GetWorkflowAggregatesInput{
    GroupByStatus: true,
    GroupByName:   true,
    StartTime:     time.Now().Add(-24 * time.Hour),
})
for _, r := range rows {
    status := *r.Group["status"]
    name := *r.Group["name"]
    log.Printf("status=%s name=%s count=%d", status, name, r.Count)
}
```

Input fields:

- Grouping flags (at least one must be true, or `TimeBucketSize > 0`):
  `GroupByStatus`, `GroupByName`, `GroupByQueueName`, `GroupByExecutorID`, `GroupByApplicationVersion`
- `TimeBucketSize time.Duration`: when non-zero, also groups by `created_at` bucket of this duration
- Filters (all optional, AND-ed together): `Status []WorkflowStatusType`, `StartTime`, `EndTime time.Time`, `Name`, `ApplicationVersion`, `ExecutorID`, `QueueName`, `WorkflowIDPrefix []string`

Each `WorkflowAggregateRow` has a `Count int64` and a `Group map[string]*string` with one entry per enabled grouping column (`"status"`, `"name"`, `"queue_name"`, `"executor_id"`, `"application_version"`, `"time_bucket"`). Map values are pointers so `nil` represents NULL grouping values (e.g. workflows without a queue name).

Time bucket example — hourly histogram of failed workflows over the last day:

```go
rows, err := dbos.GetWorkflowAggregates(ctx, dbos.GetWorkflowAggregatesInput{
    TimeBucketSize: time.Hour,
    Status:         []dbos.WorkflowStatusType{dbos.WorkflowStatusError},
    StartTime:      time.Now().Add(-24 * time.Hour),
})
```

Safe to call from inside a workflow — the call is checkpointed as the step `DBOS.getWorkflowAggregates`.

### Step Aggregates

`dbos.GetStepAggregates` returns aggregate counts and/or max durations of steps, grouped by function name and/or status, optionally bucketed by `completed_at` time:

```go
rows, err := dbos.GetStepAggregates(ctx, dbos.GetStepAggregatesInput{
    GroupByFunctionName: true,
    SelectCount:         true,
    SelectMaxDurationMs: true,
    CompletedAfter:      time.Now().Add(-24 * time.Hour),
})
for _, r := range rows {
    fmt.Printf("step=%s count=%d max_duration_ms=%d\n",
        *r.Group["function_name"], *r.Count, *r.MaxDurationMs)
}
```

- Grouping flags (at least one must be true, or `TimeBucketSize > 0`): `GroupByFunctionName`, `GroupByStatus`
- Aggregate flags (at least one must be true): `SelectCount`, `SelectMaxDurationMs`
- Filters: `Status []string`, `FunctionName []string`, `WorkflowIDPrefix []string`, `CompletedAfter`, `CompletedBefore time.Time`
- Step status is derived from the recorded outcome: no recorded error means `SUCCESS`, otherwise `ERROR`

Each `StepAggregateRow` has a `Group map[string]*string` (entries: `"function_name"`, `"status"`, `"time_bucket"`) and pointer fields `Count` and `MaxDurationMs`, populated only for the enabled `Select*` flags.

Reference: [Workflow Management](https://docs.dbos.dev/golang/tutorials/workflow-management)
