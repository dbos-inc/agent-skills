---
title: Configure Step Retries for Transient Failures
impact: HIGH
impactDescription: Automatic retries with backoff handle transient failures without custom code
tags: step, retry, backoff, resilience, StepOptions
---

## Configure Step Retries for Transient Failures

Steps run once by default. For unreliable dependencies, let DBOS retry the step with exponential backoff instead of
writing retry loops by hand — hand-rolled loops are invisible to DBOS and complicate the workflow body.

**Incorrect (manual retry loop inside the workflow):**

```java
@Workflow
public String fetchWorkflow(String url) throws Exception {
  for (int attempt = 0; attempt < 5; attempt++) {
    try {
      return dbos.runStep(() -> fetch(url), "fetch");
    } catch (Exception e) {
      Thread.sleep(1000); // blocks the workflow and is not durable
    }
  }
  throw new IllegalStateException("giving up");
}
```

**Correct (declarative retry configuration):**

```java
import dev.dbos.transact.workflow.StepOptions;

@Workflow
public String fetchWorkflow(String url) throws Exception {
  return dbos.runStep(
      () -> fetch(url),
      new StepOptions("fetch")
          .withMaxAttempts(10)                        // total attempts, default 1
          .withRetryInterval(Duration.ofMillis(500))  // delay before first retry, default 1s
          .withBackoffRate(2.0));                     // multiplier per retry, default 2.0
}
```

The same options are available on the `@Step` annotation for reusable steps:

```java
@Step(name = "fetch", maxAttempts = 10, intervalSeconds = 0.5, backOffRate = 2.0)
public String fetch(String url) throws Exception { /* ... */ }
```

To avoid burning attempts on errors that will never succeed, supply a retry predicate. Use
`withShouldRetry(Predicate<Throwable>)` with `runStep`, or a `StepShouldRetry` class with the annotation:

```java
// Lambda form
dbos.runStep(() -> fetch(url),
    new StepOptions("fetch")
        .withMaxAttempts(5)
        .withShouldRetry(e -> !(e instanceof ValidationException)));

// Annotation form — the class must be public with a public no-arg constructor
public class NoRetryOnValidation implements StepShouldRetry {
  @Override
  public boolean shouldRetry(Throwable e) {
    return !(e instanceof ValidationException);
  }
}

@Step(maxAttempts = 5, shouldRetry = NoRetryOnValidation.class)
public String fetchData(String id) { /* ... */ }
```

Behavior:

- `maxAttempts` counts total attempts, so `1` (the default) means no retries; it must be at least 1
- `retryInterval` must be positive and `backOffRate` at least 1.0
- Returning `false` from `shouldRetry` fails the step immediately without consuming remaining attempts
- When all attempts are exhausted the exception propagates to the workflow, which can catch it and take a
  compensating path
- Retries happen inside a single step checkpoint: once the step finally succeeds, only its final result is recorded

Reference: [Configurable Retries](https://docs.dbos.dev/java/tutorials/step-tutorial#configurable-retries)
