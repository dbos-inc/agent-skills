---
title: Use Streams for Real-Time Data
impact: MEDIUM
impactDescription: Enables streaming results from long-running workflows
tags: communication, stream, real-time, readStream, writeStream, readStreamOffset
---

## Use Streams for Real-Time Data

Workflows can stream data to clients in real-time using `DBOS.writeStream`, `DBOS.closeStream`, and `DBOS.readStream`. Useful for LLM output streaming or progress reporting.

**Incorrect (accumulating results then returning at end):**

```typescript
async function processWorkflowFn() {
  const results: string[] = [];
  for (const chunk of data) {
    results.push(await processChunk(chunk));
  }
  return results; // Client must wait for entire workflow to complete
}
```

**Correct (streaming results as they become available):**

```typescript
async function processWorkflowFn() {
  for (const chunk of data) {
    const result = await DBOS.runStep(() => processChunk(chunk), { name: "process" });
    await DBOS.writeStream("results", result);
  }
  await DBOS.closeStream("results"); // Signal completion
}
const processWorkflow = DBOS.registerWorkflow(processWorkflowFn);

// Read the stream from outside
const handle = await DBOS.startWorkflow(processWorkflow)();
for await (const value of DBOS.readStream<string>(handle.workflowID, "results")) {
  console.log(`Received: ${value}`);
}
```

Writing behaviors:
- A workflow may have any number of streams, each identified by a unique key
- `writeStream` and `closeStream` can be called from a workflow or its steps
- Streams are immutable and append-only
- Writes from workflows happen exactly-once; writes from steps happen at-least-once (a retried step may write duplicates)
- After `closeStream`, readers stop at the close, so values written afterward are never read
- Streams are automatically closed when the workflow terminates

### Read Options and Timeouts

```typescript
interface ReadStreamOptions {
  offset?: number;            // Start reading at this offset (default 0)
  timeoutSeconds?: number;    // Max wait for EACH value (default: wait indefinitely)
  pollingIntervalMs?: number; // Interval between system-database polls (>= 1)
}
```

`timeoutSeconds` bounds the gap between values, not the whole read: the clock restarts every time a value arrives. When it expires, the read throws `DBOSStreamTimeoutError`. Match it with `isStreamTimeoutError` from the SDK's `Error` namespace, not `instanceof` (a replayed checkpointed timeout is revived as a plain `Error`):

```typescript
import { DBOS, Error as DBOSErrors } from "@dbos-inc/dbos-sdk";

try {
  // Resume after the 10 values already consumed
  for await (const value of DBOS.readStream(workflowID, "results", { offset: 10, timeoutSeconds: 30 })) {
    console.log(value);
  }
} catch (e) {
  if (DBOSErrors.isStreamTimeoutError(e)) {
    console.log("The producer stopped sending values");
  } else {
    throw e;
  }
}
```

### Reading a Single Value

Use `DBOS.readStreamOffset` to wait for and read the one value at a given offset:

```typescript
const value = await DBOS.readStreamOffset<string>(workflowID, "results", 5, { timeoutSeconds: 30 });
```

It throws `DBOSStreamTimeoutError` if the timeout passes or if the stream ends before reaching that offset.

Reading behaviors:
- `readStream` returns an async generator that yields values in order until the stream is closed or the workflow terminates
- When called from workflow code, each value read by `readStream`/`readStreamOffset` is checkpointed as a step, so a replayed workflow re-yields the same values
- Both throw `DBOSNonExistentWorkflowError` if no workflow with that ID exists
- From outside the application, use `client.readStream` (never checkpointed) and `client.readStreamOffset` with the same options

Reference: [Workflow Streaming](https://docs.dbos.dev/typescript/tutorials/workflow-communication#workflow-streaming)
