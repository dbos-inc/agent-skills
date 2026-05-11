---
title: Manage Database-Backed Queues at Runtime
impact: HIGH
impactDescription: Inspect, reconfigure, and delete queues without redeploying
tags: queue, retrieve, delete, list, reconfigure, runtime
---

## Manage Database-Backed Queues at Runtime

Queue configuration lives in the system database, so any DBOS process or `DBOSClient` connected to the same database can inspect and reconfigure queues without restarts or redeploys.

**Incorrect (redeploying just to change a limit):**

```python
# Old approach: hardcoded in source, ship a new deploy to change.
DBOS.register_queue("email", concurrency=10)
```

**Correct (reconfigure at runtime):**

```python
# From an admin tool or a running DBOS process - no redeploy needed.
queue = DBOS.retrieve_queue("email")
queue.set_concurrency(50)
queue.set_limiter({"limit": 500, "period": 60})
```

Workers pick up the new configuration on their next polling iteration.

### Retrieving and Listing Queues

```python
queue = DBOS.retrieve_queue("email")  # Returns None if not registered
if queue is not None:
    print(queue.name, queue.concurrency, queue.worker_concurrency)

for q in DBOS.list_queues():
    print(q.name, q.concurrency)
```

### All Reconfiguration Methods

```python
queue.set_concurrency(50)           # or None to remove
queue.set_worker_concurrency(5)
queue.set_limiter({"limit": 500, "period": 60})
queue.set_priority_enabled(True)
queue.set_partition_queue(False)
queue.set_polling_interval_sec(2.0)
```

In `async` code, use the `_async` variants (`set_concurrency_async`, etc.) so the database write does not block the event loop. Reading a property like `queue.concurrency` performs a synchronous database round-trip; in async code use `await queue.get_concurrency_async()` (and the other `get_*_async` getters) instead.

**Warning:** If your application calls `DBOS.register_queue` on startup, the next process to launch can overwrite settings you applied via `set_*`. Either update the `register_queue` call to match, or pass `on_conflict="never_update"` to preserve runtime changes.

### Deleting a Queue

```python
DBOS.delete_queue("email")
```

No-op if the queue does not exist.

**Warning:** Workflows already enqueued on a deleted queue can no longer be dequeued, executed, or recovered. Cancel or drain pending workflows on the queue before deleting it.

### From a DBOSClient

The same methods are available on `DBOSClient` for external services and admin tools:

```python
client.register_queue("email", concurrency=10, on_conflict="always_update")
client.retrieve_queue("email")
client.list_queues()
client.delete_queue("email")
```

`on_conflict="update_if_latest_version"` is **not** supported on the client (clients have no application version); passing it raises `DBOSException`. The client's `on_conflict` default is `"always_update"`.

Reference: [Queues Reference](https://docs.dbos.dev/python/reference/queues)
