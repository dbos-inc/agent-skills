---
title: Use Streams for Real-Time Data
impact: MEDIUM
impactDescription: Enables real-time progress and LLM streaming
tags: streaming, write_stream, read_stream, realtime
---

## Use Streams for Real-Time Data

Workflows can stream data in real-time to clients. Useful for LLM responses, progress reporting, or long-running results.

**Incorrect (returning all data at end):**

```python
@DBOS.workflow()
def llm_workflow(prompt):
    # Client waits for entire response
    response = call_llm(prompt)
    return response
```

**Correct (streaming results):**

```python
@DBOS.workflow()
def llm_workflow(prompt):
    for chunk in call_llm_streaming(prompt):
        DBOS.write_stream("response", chunk)
    DBOS.close_stream("response")
    return "complete"

# Client reads stream
@app.get("/stream/{workflow_id}")
def stream_response(workflow_id: str):
    def generate():
        for value in DBOS.read_stream(workflow_id, "response"):
            yield value
    return StreamingResponse(generate())
```

Stream characteristics:
- Streams are immutable and append-only
- Writes from workflows happen exactly-once
- Writes from steps happen at-least-once (may duplicate on retry)
- Streams auto-close when workflow terminates

Close streams explicitly when done:

```python
@DBOS.workflow()
def producer():
    DBOS.write_stream("data", {"step": 1})
    DBOS.write_stream("data", {"step": 2})
    DBOS.close_stream("data")  # Signal completion
```

### Reading from an Offset

`DBOS.read_stream(workflow_id, key, *, offset: int = 0)` accepts an `offset`: the offset to start reading from. Defaults to 0 (start of stream). A higher offset skips that many values from the beginning of the stream.

```python
# Skip the first 10 values
for value in DBOS.read_stream(workflow_id, "response", offset=10):
    yield value
```

`DBOS.read_stream_async` additionally accepts `polling_interval_sec: Optional[float] = None`: the polling interval in seconds when waiting for new values when not using LISTEN/NOTIFY. Defaults to the configured `notification_listener_polling_interval_sec` (1.0 if not configured).

Both params also exist on the `DBOSClient` read_stream methods (the client supports `offset` but not `polling_interval_sec`).

Reference: [Workflow Streaming](https://docs.dbos.dev/python/tutorials/workflow-communication#workflow-streaming)
