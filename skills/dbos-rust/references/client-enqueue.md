---
title: Enqueue and Await Workflows from External Clients
impact: MEDIUM
impactDescription: Dispatches typed or cross-language workflows onto target queues and retrieves results safely
tags: client, enqueue, enqueue_with, EnqueueOptions, retrieve_workflow, QualifiedWorkflowName
---

## Enqueue and Await Workflows from External Clients

Because a `dbos::Client` has no local executor, every workflow it starts must target a named queue via `client.enqueue`, `client.enqueue_with`, or `client.enqueue_portable`. By leaving `EnqueueOptions::app_version` at `None` (the default), enqueued rows are automatically claimed by executors running the target application's **latest** registered version.

**Incorrect (pinning stale hardcoded app_versions or awaiting retrieve_workflow unboundedly on user input):**

```rust
use dbos::{Client, Enqueue, EnqueueOptions};

async fn fragile_client_dispatch(
    client: &Client,
    user_supplied_id: &str,
) -> dbos::Result<String> {
    // Pitfall 1: Hardcoding `app_version: Some("0.1.0")` pins the row to `"0.1.0"` workers forever;
    // once the worker fleet rolls forward to `"0.2.0"`, nobody ever dequeues this job!
    let _h = client
        .enqueue_with::<_, String, dbos::EngineOnly>(
            "generate_invoice",
            "inv-99".to_owned(),
            EnqueueOptions {
                app_version: Some("0.1.0"),
                ..EnqueueOptions::on(Enqueue::new("billing"))
            },
        )
        .await?;

    // Pitfall 2: `Client::retrieve_workflow` is infallible and does NOT verify that `user_supplied_id`
    // exists in Postgres; `result().await` polls indefinitely if the caller mistyped the ID!
    let retrieved = client.retrieve_workflow::<String, dbos::EngineOnly>(user_supplied_id);
    retrieved.result().await
}
```

**Correct (enqueuing with EnqueueOptions, checking workflow_status, and bounding result waits):**

```rust
use std::time::Duration;
use dbos::{
    Client, EngineOnly, Enqueue, EnqueueOptions, Error, QualifiedWorkflowName,
    WorkflowIdReusePolicy,
};

async fn submit_and_poll_invoice(
    client: &Client,
    idempotency_key: &str,
    tenant_id: &str,
    order_id: String,
) -> dbos::Result<String> {
    // `app_version: None` lets whichever version is currently latest claim and run the workflow
    let handle = client
        .enqueue_with::<_, String, EngineOnly>(
            QualifiedWorkflowName::new("generate_invoice"),
            order_id,
            EnqueueOptions {
                workflow_id: Some(idempotency_key),
                workflow_id_reuse_policy: WorkflowIdReusePolicy::ReturnExisting,
                timeout: Some(Duration::from_secs(300)),
                ..EnqueueOptions::on(Enqueue {
                    partition_key: Some(tenant_id),
                    ..Enqueue::new("billing")
                })
            },
        )
        .await?;

    handle.result().await
}

async fn lookup_existing_result(client: &Client, workflow_id: &str) -> dbos::Result<String> {
    // Verify the row exists first so mistyped IDs fail fast instead of polling forever
    if client.workflow_status(workflow_id).await?.is_none() {
        return Err(Error::WorkflowNotFound {
            workflow_id: workflow_id.to_owned(),
        });
    }

    let handle = client.retrieve_workflow::<String, EngineOnly>(workflow_id);
    tokio::time::timeout(Duration::from_secs(10), handle.result())
        .await
        .map_err(|_| Error::InvalidArgument {
            operation: "retrieve_workflow".into(),
            detail: "workflow is still running".to_owned(),
        })?
}
```

### `Client` Enqueue Details

- **Target queue must be registered on the workers**: Ensure the target worker application registers `"billing"` via `register_queue` (or create it from `Client::register_queue`) so dequeue workers poll it.
- **Type parameters `<P, R, E>`**: Because `Client` has no `WorkflowRef`, annotate the expected output `R` and error `E` (`dbos::EngineOnly` or a shared domain `DurableError` crate) via turbofish `client.enqueue::<_, Output, MyError>(...)` or variable type annotation `let handle: WorkflowHandle<Output, MyError> = ...`.
- **Cross-language target**: When enqueueing into a Python, TypeScript, Go, or Java worker, either set `Enqueue { serialization: SerializationFormat::Portable, .. }` on `EnqueueOptions` or call `client.enqueue_portable` with `PortableWorkflowArgs` (see [advanced-interops.md](advanced-interops.md)).

Reference: [dbos::Client::enqueue_with](https://docs.rs/dbos/latest/dbos/struct.Client.html#method.enqueue_with), [dbos::EnqueueOptions](https://docs.rs/dbos/latest/dbos/struct.EnqueueOptions.html), [dbos::Client::retrieve_workflow](https://docs.rs/dbos/latest/dbos/struct.Client.html#method.retrieve_workflow)
