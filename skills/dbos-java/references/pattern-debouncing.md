---
title: Debounce Workflows Triggered in Bursts
impact: LOW-MEDIUM
impactDescription: Collapses rapid repeated triggers into a single execution with the latest inputs
tags: pattern, debouncing, debouncer, throttling, efficiency
---

## Debounce Workflows Triggered in Bursts

A debouncer delays a workflow until a key has been quiet for a given period, then runs it once with the most recent
arguments. Use it for work triggered by user activity — reindexing a document while it is being edited, syncing a
record on every field change.

**Incorrect (running the workflow on every trigger):**

```java
// Ten keystrokes start ten reindexing workflows; nine are wasted work
onDocumentEdit(doc -> dbos.startWorkflow(() -> proxy.reindex(doc)));
```

**Correct (debounce per key):**

```java
var debouncer = dbos.<String>debouncer()
    .withDebounceTimeout(Duration.ofMinutes(5)); // absolute cap per key

// Each call restarts the 60-second inactivity window for this user.
// The workflow runs once, 60 seconds after the last call, with the latest input.
WorkflowHandle<String, Exception> handle = debouncer.debounce(
    userId,
    Duration.ofSeconds(60),
    () -> svc.processInput(userInput));

String result = handle.getResult();
```

Behavior and configuration:

- `dbos.<R>debouncer()` returns an immutable builder; each `with` method returns a new instance
- `debounce(String debounceKey, Duration debouncePeriod, lambda)` groups calls by key — different keys debounce
  independently — and returns a handle to the workflow that will eventually run
- Every call resets the inactivity window; `withDebounceTimeout(Duration)` caps how long absorbing may continue from
  the first call, after which the workflow starts regardless
- Once the workflow begins executing, the next `debounce` call starts a fresh debouncing cycle
- Other options: `withQueue(String|Queue)`, `withPriority(Integer)`, `withDeduplicationId(String)`,
  `withAppVersion(String)`
- Overloads accept a `ThrowingRunnable` for void workflows and a `ThrowingSupplier` for workflows returning a value

From outside the application, use `DBOSClient.debouncer(workflowName)`, which requires `withClassName(...)` and
takes positional arguments instead of a proxy lambda:

```java
var clientDebouncer = client.<String>debouncer("processInput")
    .withClassName(MyServiceImpl.class.getName())
    .withDebounceTimeout(Duration.ofMinutes(5));

clientDebouncer.debounce(userId, Duration.ofSeconds(60), userInput);
```

Reference: [Debouncing](https://docs.dbos.dev/java/reference/methods#debouncing)
