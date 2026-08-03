---
title: Use Streams for Real-Time Data
impact: MEDIUM
impactDescription: Enables streaming results from long-running workflows
tags: communication, stream, real-time, channel
---

## Use Streams for Real-Time Data

Workflows can stream data to clients in real-time using `dbos.WriteStream`, `dbos.CloseStream`, and `dbos.ReadStream`/`dbos.ReadStreamAsync`. Useful for LLM output streaming or progress reporting.

**Incorrect (accumulating results then returning at end):**

```go
func processWorkflow(ctx dbos.Context, items []string) ([]string, error) {
	var results []string
	for _, item := range items {
		result, _ := dbos.RunAsStep(ctx, func(ctx context.Context) (string, error) {
			return processItem(item)
		}, dbos.WithStepName("process"))
		results = append(results, result)
	}
	return results, nil // Client must wait for entire workflow to complete
}
```

**Correct (streaming results as they become available):**

```go
func processWorkflow(ctx dbos.Context, items []string) (string, error) {
	for _, item := range items {
		result, err := dbos.RunAsStep(ctx, func(ctx context.Context) (string, error) {
			return processItem(item)
		}, dbos.WithStepName("process"))
		if err != nil {
			return "", err
		}
		dbos.WriteStream(ctx, "results", result)
	}
	dbos.CloseStream(ctx, "results") // Signal completion
	return "done", nil
}

// Read the stream synchronously (blocks until closed)
handle, _ := dbos.RunWorkflow(ctx, processWorkflow, items)
values, closed, err := dbos.ReadStream[string](ctx, handle.GetWorkflowID(), "results")
```

**Async stream reading with channels:**

```go
ch, err := dbos.ReadStreamAsync[string](ctx, handle.GetWorkflowID(), "results")
if err != nil {
	log.Fatal(err)
}
for sv := range ch {
	if sv.Err != nil {
		log.Fatal(sv.Err)
	}
	if sv.Closed {
		break
	}
	fmt.Println("Received:", sv.Value)
}
```

**Snapshot reads (non-blocking):**

```go
// Returns once all currently-available values are drained
values, closed, err := dbos.ReadStream[string](ctx, workflowID, "results",
	dbos.WithReadStreamSnapshot())

// Start reading at offset 100, skipping earlier values
values, closed, err = dbos.ReadStream[string](ctx, workflowID, "results",
	dbos.WithReadStreamFromOffset(100))
```

Key behaviors:
- A workflow may have any number of streams, each identified by a unique key
- Streams are immutable and append-only
- Writes from workflows happen exactly-once
- Streams are automatically closed when the workflow terminates
- `ReadStream` blocks until the workflow is inactive or the stream is closed
- `ReadStreamAsync` returns a channel of `StreamValue[R]` for non-blocking reads
- `ReadStream` and `ReadStreamAsync` take a `dbos.Client`, so they also work from external applications using the DBOS Client (a `dbos.Context` satisfies `Client`)
- `WithReadStreamSnapshot()` makes a read return as soon as currently-available values are drained, instead of blocking until the stream closes
- `WithReadStreamFromOffset(offset)` starts the read at a 0-based offset, skipping earlier values
- `WriteStream` may be called inside a step; `CloseStream` must be called from workflow code and returns an error if called inside a step

Reference: [Workflow Streaming](https://docs.dbos.dev/golang/tutorials/workflow-communication#workflow-streaming)
