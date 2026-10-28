---
title: Tag Workflows with Searchable Attributes
impact: MEDIUM
impactDescription: Find workflows by business metadata (customer, tenant, region) without encoding it in workflow IDs
tags: workflow, attributes, WithWorkflowAttributes, SetWorkflowAttributes, WithMergeAttributes, WithFilterAttributes, metadata
---

## Tag Workflows with Searchable Attributes

Attach a JSON-serializable `map[string]any` of attributes to a workflow when you start it with `dbos.WithWorkflowAttributes`, then find workflows with the `dbos.WithFilterAttributes` filter.

**Incorrect (encoding metadata in workflow IDs and scanning):**

```go
handle, err := dbos.RunWorkflow(ctx, processOrder, order,
	dbos.WithWorkflowID("acme-us-east-1-"+uuid.NewString()))

// Fragile prefix matching, can't query by region alone
acme, err := dbos.ListWorkflows(ctx, dbos.WithFilterWorkflowIDPrefix("acme-"))
```

**Correct (attributes):**

```go
handle, err := dbos.RunWorkflow(ctx, processOrder, order,
	dbos.WithWorkflowAttributes(map[string]any{"customer": "acme", "region": "us-east-1"}))

// Matches workflows whose attributes contain ALL given key-value pairs
acme, err := dbos.ListWorkflows(ctx, dbos.WithFilterAttributes(map[string]any{"customer": "acme"}))
for _, wf := range acme {
	fmt.Println(wf.ID, wf.Attributes)
}
```

- Attributes are recorded at creation (`WorkflowStatus.Attributes`) and are **not** inherited by child workflows
- `WithFilterAttributes` requires a Postgres system database (GIN-indexed JSONB); listing fails with an error on SQLite. Nested values are matched exactly
- When enqueueing with `dbos.Enqueue` (from a `Context` or a client), set them with `dbos.WithEnqueueAttributes(map[string]any{...})`

### Updating Attributes

`dbos.SetWorkflowAttributes(ctx, workflowID, attributes, opts...)` **replaces** the whole map by default; a nil or empty map clears it. To add or update individual keys while keeping the rest, pass `dbos.WithMergeAttributes()` (v1.8.0+): the given top-level keys are merged into the existing attributes in a single update, so concurrent merges don't lose each other's keys. A matching key is overwritten, even when its value is a nested object; a nil value is stored as JSON null; merging a nil map returns an error.

**Incorrect (replace drops the other keys):**

```go
// Attributes were {"customer": "acme", "region": "us-east-1"}; now they are just {"phase": "shipping"}
err := dbos.SetWorkflowAttributes(ctx, workflowID, map[string]any{"phase": "shipping"})
```

**Correct (merge keeps the other keys):**

```go
func processOrder(ctx dbos.Context, order Order) (string, error) {
	// ... reserve inventory in a step ...
	workflowID, err := dbos.GetWorkflowID(ctx)
	if err != nil {
		return "", err
	}
	err = dbos.SetWorkflowAttributes(ctx, workflowID,
		map[string]any{"phase": "shipping"}, dbos.WithMergeAttributes())
	if err != nil {
		return "", err
	}
	// ... ship in a step ...
	return "shipped", nil
}
```

`SetWorkflowAttributes` takes any `dbos.Client`, so it also works from a standalone client. It is safe to call from within a workflow, including on itself: the update is recorded as a step, so recovery runs it once. It returns an error if the workflow does not exist.

Workflows enqueued by a schedule are tagged with its name (`WorkflowStatus.ScheduleName`); list a schedule's runs with `dbos.WithFilterScheduleName("my-schedule")`.

Reference: [Workflow Management](https://docs.dbos.dev/golang/tutorials/workflow-management)
