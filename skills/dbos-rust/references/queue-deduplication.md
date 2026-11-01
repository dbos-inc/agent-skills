---
title: Deduplicate Queued Backlog Work with deduplication_id
impact: MEDIUM
impactDescription: Prevents duplicate in-flight or waiting workflows for the same key on a queue
tags: queue, deduplication, deduplication_id, DuplicationPolicy, singleton
---

## Deduplicate Queued Backlog Work with deduplication_id

Set `Enqueue::deduplication_id` to guarantee that at most **one** active (`DELAYED`, `ENQUEUED`, or `PENDING`) workflow holds that key on the queue at any time. Once the workflow reaches a terminal status (`SUCCESS`, `ERROR`, `CANCELLED`, or `MAX_RECOVERY_ATTEMPTS_EXCEEDED`), DBOS clears its `deduplication_id` so future requests for the same key enqueue a fresh run. (For permanent all-time idempotency across completed runs too, set `StartOptions::workflow_id` instead; see [pattern-idempotency.md](pattern-idempotency.md).)

**Incorrect (setting DuplicationPolicy::ReturnExisting without a deduplication_id):**

```rust
use dbos::{DuplicationPolicy, Enqueue, StartOptions, WorkflowRef};

async fn bad_dedup_enqueue(sync_wf: &WorkflowRef<String, ()>, account: &str) -> dbos::Result<()> {
    // Rejected with `Error::InvalidArgument`: `ReturnExisting` has nothing to match when
    // `deduplication_id` is `None`!
    sync_wf
        .start_with(
            account.to_owned(),
            StartOptions {
                queue: Some(Enqueue {
                    duplication_policy: DuplicationPolicy::ReturnExisting,
                    ..Enqueue::new("account-sync")
                }),
                ..StartOptions::default()
            },
        )
        .await?;
    Ok(())
}
```

**Correct (rejecting duplicate submissions or joining the active holder):**

```rust
use dbos::{DuplicationPolicy, Enqueue, Error, StartOptions, WorkflowHandle, WorkflowRef};
use dbos::sysdb::Error as SysdbError;

// Pattern 1: Default `DuplicationPolicy::Reject` rejects concurrent duplicates explicitly
async fn enqueue_strict_once_at_a_time(
    sync_wf: &WorkflowRef<String, String>,
    account_id: &str,
) -> dbos::Result<Option<WorkflowHandle<String>>> {
    let res = sync_wf
        .start_with(
            account_id.to_owned(),
            StartOptions {
                queue: Some(Enqueue {
                    deduplication_id: Some(account_id),
                    ..Enqueue::new("account-sync")
                }),
                ..StartOptions::default()
            },
        )
        .await;

    match res {
        Ok(handle) => Ok(Some(handle)),
        Err(Error::SystemDatabase(SysdbError::QueueDeduplicated { .. })) => Ok(None),
        Err(other) => Err(other),
    }
}

// Pattern 2: `DuplicationPolicy::ReturnExisting` joins the in-flight or waiting workflow's handle
async fn enqueue_or_join_active_sync(
    sync_wf: &WorkflowRef<String, String>,
    account_id: &str,
) -> dbos::Result<String> {
    let handle = sync_wf
        .start_with(
            account_id.to_owned(),
            StartOptions {
                queue: Some(Enqueue {
                    deduplication_id: Some(account_id),
                    duplication_policy: DuplicationPolicy::ReturnExisting,
                    ..Enqueue::new("account-sync")
                }),
                ..StartOptions::default()
            },
        )
        .await?;

    handle.result().await
}
```

### Deduplication Scope Rules

- **Queue-wide across partitions**: On a partitioned queue, `deduplication_id` is enforced across the **entire queue**, not per partition. Include the partition key inside the deduplication key (`format!("{tenant_id}:{resource_id}")`) if two partitions should be allowed to hold the same logical key simultaneously.
- **Empty strings are rejected**: `deduplication_id: Some("")` fails immediately with `Error::InvalidArgument` (use `None` for no deduplication).

Reference: [dbos::Enqueue::deduplication_id](https://docs.rs/dbos/latest/dbos/struct.Enqueue.html#structfield.deduplication_id), [dbos::DuplicationPolicy](https://docs.rs/dbos/latest/dbos/enum.DuplicationPolicy.html)
