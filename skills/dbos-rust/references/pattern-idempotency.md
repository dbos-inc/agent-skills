---
title: Guarantee Exactly-Once Requests with Explicit Workflow IDs
impact: MEDIUM
impactDescription: Deduplicates retried API requests and double-submitted forms across active and historical runs
tags: pattern, idempotency, workflow_id, WorkflowIdReusePolicy, RunOptions, StartOptions
---

## Guarantee Exactly-Once Requests with Explicit Workflow IDs

Assign a caller-supplied idempotency key (from an `Idempotency-Key` HTTP header, payment intent ID, or deterministic domain key) to `workflow_id` on `RunOptions`, `StartOptions`, or `EnqueueOptions`. By default (`WorkflowIdReusePolicy::ReturnExisting`), submitting an existing `workflow_id` joins the in-flight execution or replays the stored terminal outcome without running the workflow a second time.

**Incorrect (omitting workflow_id on mutating endpoints or passing an empty string):**

```rust
use dbos::{RunOptions, WorkflowRef};

async fn unsafe_charge_handler(
    charge_wf: &WorkflowRef<u64, String>,
    idempotency_header: Option<&str>,
    cents: u64,
) -> dbos::Result<String> {
    // Bug 1: `Some("")` fails with `Error::InvalidArgument`! Map empty strings to `None` or reject 400.
    // Bug 2: When `workflow_id` is `None`, DBOS generates a fresh UUID v4 on every HTTP retry,
    // charging the customer twice!
    charge_wf
        .run_with(
            cents,
            RunOptions {
                workflow_id: idempotency_header,
                ..RunOptions::default()
            },
        )
        .await
}
```

**Correct (joining existing runs with ReturnExisting or rejecting duplicates with Reject):**

```rust
use dbos::{Error, RunOptions, StartOptions, WorkflowIdReusePolicy, WorkflowRef};
use dbos::sysdb::Error as SysdbError;

// Default `WorkflowIdReusePolicy::ReturnExisting`: safe retry-transparent HTTP POST
async fn idempotent_charge_handler(
    charge_wf: &WorkflowRef<u64, String>,
    request_key: &str,
    cents: u64,
) -> dbos::Result<String> {
    if request_key.is_empty() {
        return Err(Error::InvalidArgument {
            operation: "charge".into(),
            detail: "Idempotency-Key header must not be empty".to_owned(),
        });
    }

    // First call inserts and runs `charge_wf`; retries join the running workflow or return
    // the exact recorded output/error once settled
    charge_wf
        .run_with(
            cents,
            RunOptions {
                workflow_id: Some(request_key),
                workflow_id_reuse_policy: WorkflowIdReusePolicy::ReturnExisting,
                ..RunOptions::default()
            },
        )
        .await
}

// `WorkflowIdReusePolicy::Reject`: fail fast when a unique ID has ever been used before
async fn create_once_only(
    provision_wf: &WorkflowRef<String, String>,
    tenant_slug: &str,
) -> dbos::Result<Option<String>> {
    let key = format!("provision:{tenant_slug}");
    let res = provision_wf
        .start_with(
            tenant_slug.to_owned(),
            StartOptions {
                workflow_id: Some(&key),
                workflow_id_reuse_policy: WorkflowIdReusePolicy::Reject,
                ..StartOptions::default()
            },
        )
        .await;

    match res {
        Ok(handle) => Ok(Some(handle.workflow_id().to_owned())),
        Err(Error::SystemDatabase(SysdbError::ConflictingWorkflow { .. })) => Ok(None),
        Err(other) => Err(other),
    }
}
```

### `workflow_id` vs `Enqueue::deduplication_id`

- **`workflow_id` (all-time idempotency)**: Scoped across the entire `workflow_status` table forever (until `delete` is called). `ReturnExisting` returns the original result even days after the workflow finished, and verifies that the existing row belongs to the same `(name, class_name, config_name)` (`ConflictingWorkflow` if a different workflow already owns that ID!).
- **`Enqueue::deduplication_id` (active-backlog deduplication)**: Held only while the workflow is `DELAYED`, `ENQUEUED`, or `PENDING`, and cleared automatically on completion so the same key can run again later (see [queue-deduplication.md](queue-deduplication.md)).

Reference: [dbos::WorkflowIdReusePolicy](https://docs.rs/dbos/latest/dbos/sysdb/types/enum.WorkflowIdReusePolicy.html), [dbos::RunOptions](https://docs.rs/dbos/latest/dbos/struct.RunOptions.html), [dbos::StartOptions](https://docs.rs/dbos/latest/dbos/struct.StartOptions.html)
