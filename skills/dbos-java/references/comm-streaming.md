---
title: Stream Incremental Results from Workflows
impact: MEDIUM
impactDescription: Consumers read partial results as they are produced instead of waiting for completion
tags: communication, streaming, writeStream, readStream, progress
---

## Stream Incremental Results from Workflows

Streams let a workflow append values under a key while it runs; consumers read them in order with a blocking
iterator. Use streams for progress updates, LLM token output, or per-item results in a long batch — an event only
holds the latest value, while a stream keeps the whole ordered sequence.

**Incorrect (overwriting one event per item):**

```java
@Workflow
public void processItems(List<String> items) {
  for (String item : items) {
    // Each update clobbers the previous one — consumers miss values
    dbos.setEvent("progress", item);
  }
}
```

**Correct (append to a stream and close it):**

```java
@Workflow
public String processItems(List<String> items) {
  for (String item : items) {
    String result = dbos.runStep(() -> process(item), "process");
    dbos.writeStream("results", result);
  }
  dbos.closeStream("results"); // signals consumers there is nothing more
  return "done";
}

// Consumer — another thread, another process, or external code
Iterator<Object> it = dbos.readStream(workflowId, "results");
while (it.hasNext()) {
  System.out.println(it.next());
}
```

API and behavior:

- `writeStream(String key, Object value)` may be called from a workflow or a step; a workflow may own several
  streams distinguished by key
- `closeStream(String key)` must be called from the workflow (not a step) and tells readers no more values are
  coming
- `readStream(String workflowId, String key)` returns a blocking iterator that yields values in order and ends when
  the stream is closed or the workflow reaches a terminal state; it can be called from anywhere, including
  `DBOSClient.readStream` in an external process
- `readStream` throws `DBOSNonExistentWorkflowException` if no workflow with that ID exists — a reader that starts
  before its producer has been enqueued fails instead of blocking, so start the workflow first and read after
- On PostgreSQL, writers batch their wake-ups and push them off the write path on a short timer, so a reader in
  another process sees a value within about 10ms of the write (tune with `withNotificationCoalesceInterval`).
  A stream reader also re-queries once per second regardless, because nothing pushes the producer's termination;
  on CockroachDB and with `withUseListenNotify(false)` that same one-second poll is the only delivery path
- Values must be JSON-serializable; pass `SerializationStrategy.PORTABLE` to `writeStream` when consumers are
  written in Python or TypeScript
- Writes are checkpointed like steps, so replay does not duplicate values in the stream

Reference: [writeStream / readStream](https://docs.dbos.dev/java/reference/methods#writestream)
