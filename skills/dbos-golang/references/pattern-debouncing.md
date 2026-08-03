---
title: Debounce Workflows to Prevent Wasted Work
impact: MEDIUM
impactDescription: Prevents redundant workflow executions during rapid triggers
tags: pattern, debounce, delay, efficiency
---

## Debounce Workflows to Prevent Wasted Work

Use `dbos.NewDebouncer` to delay workflow execution until some time has passed since the last trigger. This prevents wasted work when a workflow is triggered multiple times in quick succession.

**Incorrect (executing on every trigger):**

```go
// Every keystroke triggers a new workflow - wasteful!
func onInputChange(ctx dbos.Context, userInput string) {
	dbos.RunWorkflow(ctx, processInput, userInput)
}
```

**Correct (using Debouncer):**

```go
// The target workflow must be registered before creating the debouncer
debouncer, err := dbos.NewDebouncer(ctx, processInput,
	dbos.WithDebouncerTimeout(120*time.Second), // Max wait: 2 minutes
)

func onInputChange(ctx dbos.Context, userID, userInput string) error {
	// Delays execution by 60 seconds from the last call
	// Uses the LAST set of inputs when finally executing
	_, err := debouncer.Debounce(ctx, userID, 60*time.Second, userInput)
	return err
}
```

Key behaviors:
- First argument to `Debounce` is the debounce key, grouping executions together (e.g., per user)
- Second argument is the delay duration from the last call
- `WithDebouncerTimeout` sets a max wait time since the first trigger
- When the workflow finally executes, it uses the **last** set of inputs
- After execution begins, the next `Debounce` call starts a new cycle
- `WithDebouncerQueue("name")` runs the debounced workflow on a specific queue (must already be registered)

Type signature: `Debouncer[R any, P any]` — result type first, then the workflow input type.

To debounce a workflow method of a configured instance (registered with `WithInstance`), pass the instance with `WithDebouncerInstance`:

```go
debouncer, err := dbos.NewDebouncer(ctx, slack.Send, dbos.WithDebouncerInstance(slack))
```

From an external application, use `dbos.NewDebouncerClient[R, P](workflowName, client, opts...)` — type parameters are explicit since there is no function to infer them from. For instance workflows pass the config name with `WithDebouncerConfigName("slack")`; for workflows other runtimes resolve by class name, set it with `WithDebouncerClassName`.

Reference: [Debouncing Workflows](https://docs.dbos.dev/golang/tutorials/workflow-tutorial#debouncing)
