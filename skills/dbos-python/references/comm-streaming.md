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
- `write_stream` / `close_stream` can be called from a workflow or its steps (use `write_stream_async` / `close_stream_async` in coroutine workflows)
- Streams auto-close when the workflow terminates; after `close_stream`, readers stop at the close

Close streams explicitly when done:

```python
@DBOS.workflow()
def producer():
    DBOS.write_stream("data", {"step": 1})
    DBOS.write_stream("data", {"step": 2})
    DBOS.close_stream("data")  # Signal completion
```

### Read Parameters

```python
DBOS.read_stream(
    workflow_id: str,
    key: str,
    *,
    offset: int = 0,                              # skip this many values
    polling_interval_sec: Optional[float] = None, # when not using LISTEN/NOTIFY; min 0.001
    timeout_seconds: Optional[float] = None,      # max wait for EACH value; None = forever
) -> Generator[Any, Any, None]
```

The same parameters exist on `DBOS.read_stream_async` (async generator) and on `DBOSClient.read_stream` / `read_stream_async`. `polling_interval_sec` defaults to the configured `notification_listener_polling_interval_sec` (1.0) for `DBOS` reads, and to `1.0` for `DBOSClient` reads.

**Incorrect (reader hangs forever if the producer stalls):**

```python
for value in DBOS.read_stream(workflow_id, "response"):
    yield value
```

**Correct (bound the gap between values):**

```python
from dbos import error as dboserror

try:
    for value in DBOS.read_stream(workflow_id, "response", timeout_seconds=30):
        yield value
except dboserror.DBOSStreamTimeoutError:
    ...  # producer stopped sending values
```

`timeout_seconds` restarts every time a value arrives, so it bounds the gap between values, not the total read. Reading a stream of a nonexistent workflow raises `DBOSNonExistentWorkflowError`.

### Reading a Single Offset

Use `DBOS.read_stream_offset` (or `_async`, or the client equivalents) to wait for one specific value, e.g. to resume where a previous reader left off:

```python
value = DBOS.read_stream_offset(workflow_id, "response", 5, timeout_seconds=30)
```

It raises `DBOSStreamTimeoutError` if the timeout passes or if the stream closes before reaching that offset.

### Reading From a Workflow

When `read_stream` / `read_stream_offset` is called from workflow code, each value read is checkpointed as a step, so a recovered workflow re-yields the values it originally read. `DBOSClient` reads are never checkpointed.

Reference: [Workflow Streaming](https://docs.dbos.dev/python/tutorials/workflow-communication#workflow-streaming)
