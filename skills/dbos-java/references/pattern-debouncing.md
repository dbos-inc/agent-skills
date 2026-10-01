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
- The key is released when the delay expires and the workflow becomes `ENQUEUED`, not when it starts running. The
  next `debounce` call after that starts a fresh debouncing cycle, even if the first workflow is still waiting on a
  busy queue
- Other options: `withQueue(String)` / `withQueue(QueueName)` (the `Queue` overload is deprecated for removal),
  `withPriority(Integer)`, `withAppVersion(String)`, and `withTimeout(Duration)` (1.2+), a timeout for every workflow
  the debouncer starts, counted from when it is dequeued. It takes precedence over a timeout set with
  `WorkflowOptions` around the `debounce` call, which still applies when `withTimeout` is not set. The debounced
  workflow never inherits the calling workflow's timeout or deadline
- Do not point `withQueue` at a partitioned queue: a debounced workflow has no partition key, and partitioned queues
  reject deduplication IDs
- `withPriority` requires a queue: `debounce()` throws `IllegalArgumentException` if a priority is set without
  `withQueue`. A negative priority, or a zero or negative `withTimeout`, throws `IllegalArgumentException` from the
  setter itself
- `withDeduplicationId` is deprecated for removal and ignored since 1.2 — the debounce owns the workflow's
  deduplication ID; do not use it
- Overloads accept a `ThrowingRunnable` for void workflows and a `ThrowingSupplier` for workflows returning a value
- The lambda's workflow must be registered; an unregistered one throws `IllegalStateException` from `debounce()`
- Workflows on named instances can be debounced: the call through the instance's proxy carries its instance name
  (`DebouncerClient` takes it with `withInstanceName`)
- A debounce key is held through the deduplication index on the target queue, which is shared by every application
  on a system database. If the key is held by another application's workflow, or by a workflow that is not a
  debounce of this one, `debounce()` throws `DBOSQueueDuplicatedException`

Under the hood (1.2+), the first call enqueues the workflow itself in the `DELAYED` state on its queue — the one set
with `withQueue`, or the DBOS internal queue — holding `workflowName-key` as its deduplication ID. Each later call
on the key updates that one row: it resets the start to one period after this call (never past the debounce
timeout) and replaces the arguments. When the delay expires the workflow leaves `DELAYED`, frees the key, and runs with the latest
arguments. `WorkflowStatus.isDebounced()` and `debounceDeadline()` report the debounce on that workflow.

1.1 debounced through an internal `debouncerWorkflow` instead. 1.2 still coalesces with a 1.1 `debouncerWorkflow`
that holds a key, and takes it over if it stops answering. For which releases can share a fleet, see
[lifecycle-config.md](lifecycle-config.md).

From outside the application, use `DBOSClient.debouncer(workflowName)`, which requires `withClassName(...)` and
takes positional arguments instead of a proxy lambda:

```java
var clientDebouncer = client.<String>debouncer("processInput")
    .withClassName(MyServiceImpl.class.getName())
    .withDebounceTimeout(Duration.ofMinutes(5));

clientDebouncer.debounce(userId, Duration.ofSeconds(60), userInput);
```

`DebouncerClient` also takes `withInstanceName`, `withQueue`, `withPriority`, `withAppVersion`, `withTimeout`,
`withAttributes`, and `withSerialization(SerializationStrategy)`, which should match the strategy the workflow is
registered with (for example `PORTABLE`). The same queue, priority and timeout rules apply, as does
`DBOSQueueDuplicatedException` for a key held by another application or workflow; `debounce()` throws
`IllegalStateException` if `withClassName` was not called.

Reference: [Debouncing](https://docs.dbos.dev/java/reference/methods#debouncing)
