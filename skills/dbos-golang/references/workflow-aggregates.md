---
title: Aggregate Workflow Counts for Analytics
impact: MEDIUM
impactDescription: Enables low-cost analytics over workflow status without scanning the workflow table
tags: workflow, aggregates, analytics, observability
---

## Aggregate Workflow Counts for Analytics

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

Reference: [Workflow Management](https://docs.dbos.dev/golang/tutorials/workflow-management)
