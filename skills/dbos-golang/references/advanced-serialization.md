---
title: Customize Workflow Serialization
impact: MEDIUM
impactDescription: Switch the on-disk encoding for workflow inputs, outputs, and events
tags: advanced, serialization, gob, json
---

## Customize Workflow Serialization

DBOS serializes every workflow input, step output, event payload, and message to a string in the system database. The default serializer is JSON. Override it via `Config.Serializer` (or `ClientConfig.Serializer`) when JSON cannot round-trip your types — most commonly when fields are interfaces or use Go-specific types like `time.Time` precision, `map[string]any` with mixed value types, or types implementing `gob.GobEncoder`.

### Built-in gob serializer

**Incorrect (JSON loses Go-specific types):**

```go
type Event struct {
    Payload any // interface; JSON forgets the concrete type on decode
}
// Default JSON serializer turns Event.Payload into a map[string]any after
// the workflow recovers — every step downstream now reads the wrong type.
ctx, _ := dbos.NewDBOSContext(context.Background(), dbos.Config{
    AppName:     "my-app",
    DatabaseURL: os.Getenv("DBOS_SYSTEM_DATABASE_URL"),
})
```

**Correct (switch to gob and register the concrete types):**

```go
import "encoding/gob"

func init() {
    // Register every concrete type that will cross a workflow boundary
    gob.Register(MyWorkflowInput{})
    gob.Register(MyStepOutput{})
}

ctx, _ := dbos.NewDBOSContext(context.Background(), dbos.Config{
    AppName:     "my-app",
    DatabaseURL: os.Getenv("DBOS_SYSTEM_DATABASE_URL"),
    Serializer:  dbos.NewGobSerializer(),
})
```

All workflow inputs/outputs, step outputs, events, and messages are now gob-encoded. Each workflow row records which serializer wrote it (`WorkflowStatus.Serialization`), so decoding picks the correct codec even if the active config changes later.

### Custom serializer

Implement the `Serializer[T]` interface:

```go
type Serializer[T any] interface {
    Name() string                  // unique, persisted with each workflow row
    Encode(data T) (*string, error)
    Decode(data *string) (T, error)
}
```

`Name()` must be stable for the lifetime of the database — workflows written under one name can only be decoded by a serializer that returns the same name. `nil` inputs/outputs must round-trip to `nil` (`Encode(nil)` should return `(nil, nil)`).

Pass an instance of `Serializer[any]` via `Config.Serializer`. Custom serializers are looked up by `Name()` at decode time, so register the same instance on every executor that recovers workflows from this database.

### Cross-runtime workflows

The constant `dbos.PortableSerializerName` (`"portable_json"`) marks workflows produced by the portable cross-language format. Do **not** assign this name to a custom serializer — it is reserved. See [advanced-interops.md](advanced-interops.md) for interop with Python/TypeScript workflows.

Reference: [DBOS Configuration](https://docs.dbos.dev/golang/reference/configuration)
