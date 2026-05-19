---
title: Cross-Language Workflows with Portable JSON
impact: LOW
impactDescription: Lets a Go executor enqueue, recover, and consume workflows owned by Python or TypeScript runtimes (and vice versa)
tags: advanced, interop, portable, cross-language
---

## Cross-Language Workflows with Portable JSON

When a single DBOS deployment mixes Go executors with Python or TypeScript executors, workflows must be serialized in a format every runtime understands. DBOS provides a "portable JSON" envelope for this.

### Enqueueing a foreign workflow from Go

**Incorrect (passing a Go struct to a Python workflow):**

```go
// PyInput is a Go-only struct; Python cannot decode the resulting JSON
// and the workflow will fail on dequeue.
handle, _ := dbos.Enqueue[PyInput, any](client, "task_queue", "py_workflow", PyInput{...})
```

**Correct (PortableWorkflowArgs envelope):**

To enqueue a workflow that lives in another runtime (e.g. a Python class-based workflow), pass a `PortableWorkflowArgs` envelope as the input. The client switches to portable JSON automatically when it sees this type:

```go
args := dbos.PortableWorkflowArgs{
    PositionalArgs: []any{"hello", 42},
    NamedArgs:      map[string]any{"key": "value"},
}

handle, err := dbos.Enqueue[dbos.PortableWorkflowArgs, any](
    client, "task_queue", "py_workflow", args,
    dbos.WithEnqueueClassName("MyPyClass"),       // optional
    dbos.WithEnqueueConfigName("default"),        // optional
)
```

### Running a portable workflow in Go

When a workflow is dequeued or recovered, DBOS reads its stored serialization (`WorkflowStatus.Serialization`) and adds `WithPortableWorkflow()` for you. You only set it manually when synthesizing a portable workflow run from scratch:

```go
handle, err := dbos.RunWorkflow(ctx, fn, input, dbos.WithPortableWorkflow())
```

Inside a portable workflow, all inputs, step outputs, events, messages, and streams are encoded as portable JSON regardless of the configured `Config.Serializer`.

### Cross-language errors

Use `PortableWorkflowError` to return structured error information that other runtimes can deserialize into their native exception types:

```go
return nil, &dbos.PortableWorkflowError{
    Name:    "ValidationError",
    Message: "input must be positive",
    Code:    400,
    Data:    map[string]any{"field": "amount"},
}
```

Plain `error` values still work — they round-trip as a portable error named `"Portable Error"` carrying the error string.

### Stored format

The constant `dbos.PortableSerializerName` (`"portable_json"`) identifies portable-serialized rows. Decoders dispatch on this name automatically, so:

- A Go executor can recover a workflow originally started by Python.
- Reserving the name `"portable_json"` for a custom `Serializer[any]` is not allowed — see [advanced-serialization.md](advanced-serialization.md).

Reference: [DBOS Cross-Language Workflows](https://docs.dbos.dev/)
