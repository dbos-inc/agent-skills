---
title: Share a System Database Between Applications
impact: LOW
impactDescription: Lets multiple applications (in any language) share one system database, isolated by application name, while deliberately calling each other's workflows
tags: advanced, application-name, shared-database, ownership, rename, enqueue_workflow_with_options
---

## Share a System Database Between Applications

Multiple DBOS applications, potentially in different languages, can share a single system database. Each application is identified by its configured `name` and owns everything it creates: workflows, steps, queues, schedules, and application versions. Applications are isolated by default but can interoperate by naming each other.

Ownership determines which application runs what:

- A workflow is dequeued, run, and recovered only by the application that owns it.
- A queue is polled only by the application that registered it, even if another application enqueues workflows on it.
- A schedule is fired only by the application that created it, and its workflows are owned by that application.
- Application versions are tracked per application, so one application's deployments do not affect which version its peers consider latest.

Queue, schedule, and version names remain globally unique across the shared database; registering a name another application owns raises an error. Workflow IDs are also global, so ID-addressed operations (`retrieve_workflow`, `send`, `get_event`, `read_stream`, ...) work across applications regardless of ownership. Listing operations (`list_workflows`, `list_queues`, `list_schedules`) default to the calling application's rows; pass `application_name=` (a name or list) to list others.

### Calling Another Application's Workflows

**Incorrect (enqueueing a foreign workflow without naming its owner):**

```python
from dbos import DBOS

# The enqueued workflow is owned, and only dequeued, by the calling
# application, which does not implement process_order.
handle = DBOS.enqueue_workflow_with_options(
    {"workflow_name": "process_order", "queue_name": "orders"}, "order-123"
)
```

**Correct (naming the owning application):**

```python
from dbos import DBOS, EnqueueOptions

options: EnqueueOptions = {
    "workflow_name": "process_order",
    "queue_name": "orders",
    # The application that implements process_order owns, dequeues, and runs it
    "application_name": "order-service",
}
handle = DBOS.enqueue_workflow_with_options(options, "order-123")
result = handle.get_result()  # workflow IDs are global, so waiting works
```

`DBOS.enqueue_workflow_with_options` (and `_async`) enqueues by name without a function reference, takes the same `EnqueueOptions` as `DBOSClient.enqueue`, and is safe to call from inside a workflow (the enqueued workflow is recorded as a child). Leave `app_version` unset so it runs on the owning application's latest version. If the applications are written in different languages, also set `"serialization_type": WorkflowSerializationFormat.PORTABLE` (see [advanced-serialization](advanced-serialization.md)).

### Clients Must Name Their Application

A `DBOSClient` with no `application_name` sees every application's rows, but everything it creates is owned by **no** application. Unowned rows are treated as everyone's: any application may dequeue an unowned workflow (claiming it), and every application fires unowned schedules and polls unowned queues. Always set `application_name` on clients when the system database is shared:

```python
client = DBOSClient(
    system_database_url=os.environ["DBOS_SYSTEM_DATABASE_URL"],
    application_name="order-service",  # act on behalf of this application
)
```

Per-call overrides: `application_name` in `EnqueueOptions`; on `client.register_queue`, `client.create_schedule` (and `client.apply_schedules` entries), and `set_latest_application_version`; and on `Debouncer.create` / `DebouncerClient`.

### Renaming an Application

Ownership is recorded under the application's name, so renaming requires transferring ownership of its rows. Stop the application first (a running application would race the rename, creating new work under its old name), then:

```python
counts = client.rename_application("old-name", "new-name")
# counts: {"queues": ..., "schedules": ..., "versions": ..., "workflows": ..., "steps": ...}
```

Or with the CLI: `dbos rename-application --from old-name --to new-name -s $DBOS_SYSTEM_DATABASE_URL` (or `dbosctl sysdb rename-application`). Pass `-y` to skip the `dbos` confirmation prompt; `dbosctl` requires `--force` when running non-interactively. `rename_application` is idempotent: if interrupted, re-running it resumes where it left off.

Rows created before upgrading to a version with application ownership (or by clients without a name) are unowned. Before adding a second application to an existing system database, adopt them into the first application:

```python
client.rename_application(None, "my-app", adopt_unclaimed_rows=True)
# CLI: dbos rename-application --to my-app --adopt-unclaimed-rows -s $DBOS_SYSTEM_DATABASE_URL
```

Reference: [Sharing a System Database](https://docs.dbos.dev/explanations/sharing-a-system-database)
