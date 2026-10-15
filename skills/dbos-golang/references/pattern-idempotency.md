---
title: Use Workflow IDs for Idempotency
impact: MEDIUM
impactDescription: Prevents duplicate side effects like double payments
tags: pattern, idempotency, workflow-id, deduplication, reuse-policy, authenticated-user
---

## Use Workflow IDs for Idempotency

Assign a workflow ID to ensure a workflow executes only once, even if called multiple times. This prevents duplicate side effects like double payments.

**Incorrect (no idempotency):**

```go
func processPayment(ctx dbos.Context, orderID string) (string, error) {
	_, err := dbos.RunAsStep(ctx, func(ctx context.Context) (string, error) {
		return chargeCard(orderID)
	}, dbos.WithStepName("chargeCard"))
	return "charged", err
}

// Multiple calls could charge the card multiple times!
dbos.RunWorkflow(ctx, processPayment, "order-123")
dbos.RunWorkflow(ctx, processPayment, "order-123") // Double charge!
```

**Correct (with workflow ID):**

```go
func processPayment(ctx dbos.Context, orderID string) (string, error) {
	_, err := dbos.RunAsStep(ctx, func(ctx context.Context) (string, error) {
		return chargeCard(orderID)
	}, dbos.WithStepName("chargeCard"))
	return "charged", err
}

// Same workflow ID = only one execution
workflowID := fmt.Sprintf("payment-%s", orderID)
dbos.RunWorkflow(ctx, processPayment, "order-123",
	dbos.WithWorkflowID(workflowID),
)
dbos.RunWorkflow(ctx, processPayment, "order-123",
	dbos.WithWorkflowID(workflowID),
)
// Second call returns a handle to the first execution
```

Workflow IDs must be **globally unique** for your application. If not set, a random UUID is generated. By default (`dbos.WorkflowIDReusePolicyReturnExisting`), starting a workflow with an ID already in use returns a handle to the existing workflow, whatever its status, without starting a new one; the new input is ignored and `GetResult` returns the recorded result.

### Rejecting reused workflow IDs

To detect duplicates instead of silently attaching to the existing workflow, pass `WithWorkflowIDReusePolicy(dbos.WorkflowIDReusePolicyReject)` (v1.6.0+). The start returns an error matching `dbos.ErrWorkflowIDInUse` without starting a new workflow or modifying the existing one:

```go
handle, err := dbos.RunWorkflow(ctx, processPayment, order,
	dbos.WithWorkflowID(fmt.Sprintf("payment-%s", order.ID)),
	dbos.WithWorkflowIDReusePolicy(dbos.WorkflowIDReusePolicyReject),
)
if errors.Is(err, dbos.ErrWorkflowIDInUse) {
	// A workflow with this ID already exists; its WorkflowID and RecordedName
	// are on the *dbos.Error
	return "already submitted", nil
}
```

From a client, use `WithEnqueueWorkflowIDReusePolicy(dbos.WorkflowIDReusePolicyReject)` with `dbos.Enqueue` ([client-enqueue.md](client-enqueue.md)). When a workflow starts a child with the reject policy, the rejection is checkpointed as the child-start step's outcome, so a replay returns it again. The reject policy cannot be used with a debouncer. The reuse policy is about workflow IDs; `WithDeduplicationPolicy` is about queue deduplication IDs ([queue-deduplication.md](queue-deduplication.md)).

### Reading workflow identity inside a workflow

```go
func myWorkflow(ctx dbos.Context, input string) (string, error) {
	currentID, err := dbos.GetWorkflowID(ctx)
	if err != nil {
		return "", err
	}
	// The user set with WithAuthenticatedUser (or WithEnqueueAuthenticatedUser),
	// or inherited from the parent workflow; "" if none (v1.6.0+)
	user, err := dbos.GetAuthenticatedUser(ctx)
	fmt.Printf("Running workflow %s for %s\n", currentID, user)
	return input, nil
}
```

`GetAuthenticatedUser` can be called from a workflow or from one of its steps. `GetWorkflowID`, `GetStepID`, and `GetAuthenticatedUser` all return an error outside a workflow context.

Reference: [Workflow IDs and Idempotency](https://docs.dbos.dev/golang/tutorials/workflow-tutorial#workflow-ids-and-idempotency)
