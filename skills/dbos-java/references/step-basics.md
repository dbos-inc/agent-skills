---
title: Use Steps for External and Non-Deterministic Operations
impact: HIGH
impactDescription: Steps enable recovery by checkpointing results
tags: step, runStep, external, api, checkpoint
---

## Use Steps for External and Non-Deterministic Operations

Any operation that calls an external service, touches the file system, or is otherwise non-deterministic must run as
a step so its result is checkpointed. DBOS offers two forms: `dbos.runStep(lambda, name)` for one-off work, and a
`@Step`-annotated method for logic reused across workflows.

**Incorrect (external call directly in the workflow body):**

```java
@Workflow
public String myWorkflow(String url) throws Exception {
  // Not checkpointed — re-executed in full every time the workflow recovers
  HttpResponse<String> response = HttpClient.newHttpClient()
      .send(HttpRequest.newBuilder().uri(URI.create(url)).build(),
            HttpResponse.BodyHandlers.ofString());
  return response.body();
}
```

**Correct (wrapped in a step):**

```java
class ExampleImpl implements Example {
  private final DBOS dbos;
  private Example self; // proxy for @Step calls

  ExampleImpl(DBOS dbos) { this.dbos = dbos; }

  void setSelf(Example self) { this.self = self; }

  @Override
  @Workflow
  public String myWorkflow(String url) throws Exception {
    // Inline lambda step
    String body = dbos.runStep(() -> fetch(url), "fetch");
    // Reusable @Step method — invoked through the proxy so it is checkpointed
    return self.parse(body);
  }

  @Override
  @Step
  public String parse(String body) {
    return body.strip();
  }

  private String fetch(String url) throws Exception {
    HttpResponse<String> response = HttpClient.newHttpClient()
        .send(HttpRequest.newBuilder().uri(URI.create(url)).build(),
              HttpResponse.BodyHandlers.ofString());
    return response.body();
  }
}
```

`runStep` overloads:

```java
<T, E extends Exception> T runStep(ThrowingSupplier<T, E> step, String name) throws E
<T, E extends Exception> T runStep(ThrowingSupplier<T, E> step, StepOptions opts) throws E
<E extends Exception> void runStep(ThrowingRunnable<E> step, String name) throws E
<E extends Exception> void runStep(ThrowingRunnable<E> step, StepOptions opts) throws E
```

Step requirements:

- Every step needs a name; it is recorded in the database and used for tracing (steps are identified by their
  position in the workflow, so names need not be unique)
- Step return values must be JSON-serializable
- Steps must not start, enqueue, or await workflows, and must not call `send`, `recv`, `setEvent`, or `getEvent`
- A `@Step` method is only checkpointed when called through the registered proxy (or, in Spring Boot, an injected
  self-reference); calling it via `this` runs it as a plain method
- Calling a step from inside another step folds it into the outer step's execution
- Steps run once by default; configure retries via [step-retries.md](step-retries.md)
- For steps that write to your own PostgreSQL database, prefer a transactional step so the write and the checkpoint
  commit atomically ([step-transactions.md](step-transactions.md))

Reference: [DBOS Steps](https://docs.dbos.dev/java/tutorials/step-tutorial)
