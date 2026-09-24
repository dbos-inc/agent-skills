---
title: Set Step Timeouts and Honor Abort Signals
impact: HIGH
impactDescription: Prevents hung external calls from stalling workflows and lets steps stop promptly on cancellation
tags: step, timeout, timeoutMS, AbortSignal, cancelSignal, timeoutSignal
---

## Set Step Timeouts and Honor Abort Signals

Set `timeoutMS` in a step's config to bound each attempt. An attempt that runs longer fails with `DBOSStepTimeoutError`; if `retriesAllowed` is `true`, it is retried like any other failure. Timeouts are **cooperative**: DBOS does not kill the step. It aborts `DBOS.stepStatus.timeoutSignal` and discards the result of a step that keeps running.

**Incorrect (hung call with no timeout, signal ignored):**

```typescript
async function fetchData() {
  // If the server hangs, this step (and its workflow) hangs forever
  return await fetch("https://example.com").then((r) => r.text());
}

async function workflowFn() {
  return await DBOS.runStep(fetchData, { name: "fetchData" });
}
```

**Correct (timeout plus the abort signal):**

```typescript
async function fetchData() {
  // fetch aborts as soon as this attempt's timeout fires
  const response = await fetch("https://example.com", { signal: DBOS.stepStatus?.timeoutSignal });
  return await response.text();
}

async function workflowFn() {
  return await DBOS.runStep(fetchData, {
    name: "fetchData",
    timeoutMS: 5000,
    retriesAllowed: true, // a timed-out attempt is retried
    maxAttempts: 3,
  });
}
```

`timeoutMS` works the same on `runStep`, `registerStep`, and `@DBOS.step()`. A fresh `timeoutSignal` is issued for each retry attempt. It only applies when the step is called from a workflow (outside a workflow, steps run as plain calls with no timeout).

### Stopping on Workflow Cancellation (5.1+)

Cancelling a workflow (or a workflow timeout) doesn't interrupt a running step; the workflow stops at the start of its next step. `DBOS.stepStatus.cancelSignal` fires (within about a second) when the step's workflow is cancelled, so the step can stop early. Combine it with `timeoutSignal` using `AbortSignal.any`:

```typescript
async function fetchData() {
  // timeoutSignal is set because this step runs with timeoutMS
  const { timeoutSignal, cancelSignal } = DBOS.stepStatus!;
  const signal = AbortSignal.any([timeoutSignal!, cancelSignal]);
  const response = await fetch("https://example.com", { signal });
  return await response.text();
}

async function workflowFn() {
  return await DBOS.runStep(fetchData, { name: "fetchData", timeoutMS: 5000 });
}
```

`timeoutSignal` is `undefined` unless the step has `timeoutMS`; `cancelSignal` is always present inside a step.

Reference: [Step Timeouts](https://docs.dbos.dev/typescript/tutorials/step-tutorial#step-timeouts)
