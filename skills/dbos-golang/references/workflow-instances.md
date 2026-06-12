---
title: Register Workflow Methods on Configured Instances
impact: MEDIUM
impactDescription: Enables per-instance workflow methods with correct recovery
tags: workflow, instance, configured, method, registration
---

## Register Workflow Methods on Configured Instances

To use a struct method as a workflow, register it with `dbos.WithInstance`. Method values bound to different receivers (e.g. `a.Send` and `b.Send`) share a function name, so each instance must be registered under a per-instance key derived from its config name.

The instance must implement the `ConfiguredInstance` interface:

```go
type ConfiguredInstance interface {
	ConfigName() string
}
```

**Incorrect (registering bound methods directly):**

```go
slack := &Messenger{name: "slack"}
email := &Messenger{name: "email"}
// Both methods register under the same function name - collision
dbos.RegisterWorkflow(ctx, slack.Send)
dbos.RegisterWorkflow(ctx, email.Send)
```

**Correct (registering with WithInstance):**

```go
type Messenger struct {
	name string
}

func (m *Messenger) ConfigName() string {
	return m.name
}

func (m *Messenger) Send(ctx dbos.DBOSContext, message string) (string, error) {
	// Workflow implementation using m...
	return "sent", nil
}

slack := &Messenger{name: "slack"}
email := &Messenger{name: "email"}

dbos.RegisterWorkflow(ctx, slack.Send, dbos.WithInstance(slack))
dbos.RegisterWorkflow(ctx, email.Send, dbos.WithInstance(email))
```

Run an instance workflow with the matching `WithRunInstance` option:

```go
handle, err := dbos.RunWorkflow(ctx, slack.Send, "hello", dbos.WithRunInstance(slack))
```

Key behaviors:

- `ConfigName()` must return a stable, unique name: it is durably recorded so recovery runs the workflow on the correct instance
- Instances must be registered with the same config name on every process start, before `Launch()`
- When enqueueing from the DBOS Client, pass the instance's config name with `WithEnqueueConfigName`
- To debounce an instance method, pass `WithDebouncerInstance(instance)` to `NewDebouncer` (or `WithDebouncerConfigName(name)` to `NewDebouncerClient`)

Reference: [Workflow Registration](https://docs.dbos.dev/golang/reference/workflows-steps#withinstance)
