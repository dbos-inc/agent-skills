---
title: Configure Step Retries for Transient Failures
impact: HIGH
impactDescription: Automatic retries handle transient failures without manual code
tags: step, retry, exponential-backoff, resilience, shouldRetry
---

## Configure Step Retries for Transient Failures

Steps can automatically retry on failure with exponential backoff. This handles transient failures like network issues.

**Incorrect (manual retry logic):**

```typescript
async function fetchData() {
  for (let attempt = 0; attempt < 3; attempt++) {
    try {
      return await fetch("https://api.example.com").then(r => r.json());
    } catch (e) {
      if (attempt === 2) throw e;
      await new Promise(r => setTimeout(r, 2 ** attempt * 1000));
    }
  }
}
```

**Correct (built-in retries with `DBOS.runStep`):**

```typescript
async function fetchData() {
  return await fetch("https://api.example.com").then(r => r.json());
}

async function myWorkflowFn() {
  const data = await DBOS.runStep(fetchData, {
    name: "fetchData",
    retriesAllowed: true,
    maxAttempts: 10,
    intervalSeconds: 1,
    backoffRate: 2,
  });
}
const myWorkflow = DBOS.registerWorkflow(myWorkflowFn);
```

With an inline arrow function:

```typescript
async function myWorkflowFn() {
  const data = await DBOS.runStep(
    () => fetch("https://api.example.com").then(r => r.json()),
    { name: "fetchData", retriesAllowed: true, maxAttempts: 10 }
  );
}
```

Retry parameters:
- `retriesAllowed`: Enable automatic retries (default: `false`)
- `maxAttempts`: Maximum retry attempts (default: `3`)
- `intervalSeconds`: Initial delay between retries in seconds (default: `1`)
- `backoffRate`: Multiplier for exponential backoff (default: `2`)
- `shouldRetry`: Optional predicate for selective retries (see below)

With defaults, retry delays are: 1s, 2s, 4s, 8s, 16s...

If all retries are exhausted, a `DBOSMaxStepRetriesError` is thrown to the calling workflow.

### Filtering Retries With `shouldRetry`

By default every error is retried. Use `shouldRetry` to skip retries for non-transient errors like validation failures or 4xx responses. The predicate receives the thrown error; returning `false` (or a Promise resolving to `false`) re-throws it immediately without further retries.

**Incorrect (retrying non-transient errors wastes attempts):**

```typescript
await DBOS.runStep(
  async () => {
    const r = await fetch(`https://api.example.com/users/${userId}`);
    if (!r.ok) throw new HTTPError(r.status);  // 404 retries 10 times before failing
    return r.json();
  },
  { name: "fetchUser", retriesAllowed: true, maxAttempts: 10 }
);
```

**Correct (only retry transient failures):**

```typescript
class HTTPError extends Error {
  constructor(public status: number) { super(`HTTP ${status}`); }
}

await DBOS.runStep(
  async () => {
    const r = await fetch(`https://api.example.com/users/${userId}`);
    if (!r.ok) throw new HTTPError(r.status);
    return r.json();
  },
  {
    name: "fetchUser",
    retriesAllowed: true,
    maxAttempts: 10,
    shouldRetry: (e) => !(e instanceof HTTPError && e.status >= 400 && e.status < 500),
  }
);
```

The predicate may be async, and works identically with `runStep`, `registerStep`, and `@DBOS.step`:

```typescript
async function isRetryable(error: unknown): Promise<boolean> {
  return !(error instanceof FatalError);
}

class Example {
  @DBOS.step({ retriesAllowed: true, maxAttempts: 10, shouldRetry: isRetryable })
  static async fetchData() {
    // ...
  }
}
```

Notes:
- If the predicate itself throws or rejects, that error becomes the step's failure and propagates to the workflow.
- `shouldRetry` is ignored when `retriesAllowed` is `false`.

Reference: [Configurable Retries](https://docs.dbos.dev/typescript/tutorials/step-tutorial#configurable-retries)
