---
title: Manage Workflows with Cancel, Resume, Fork, Rewind, and Delete
impact: MEDIUM
impactDescription: Safely pauses, recovers, branches, re-runs, and cleans up workflows using DBOS Rust's bare-verb management methods
tags: workflow, cancel, resume, fork, rewind, delete, management
---

## Manage Workflows with Cancel, Resume, Fork, Rewind, and Delete

On both `DBOS` and `Client`, workflow lifecycle management uses **bare verbs** (`cancel`, `resume`, `fork`, `rewind`, `delete`) and `_all` bulk variants (`cancel_all`, `resume_all`, `fork_all`, `delete_all`), rather than `cancel_workflow` or `fork_workflow`. Choose between `resume` (continue from last checkpoint), `fork` (branch into a **new** `workflow_id` from `start_step`), and `rewind` (re-run a finished workflow **in place** under the **same** `workflow_id` from `start_step`).

**Incorrect (resuming a potentially live workflow without cancelling, or omitting partition key on fork):**

```rust
use dbos::{DBOS, Fork, ForkOptions};

async fn unsafe_recovery(dbos: &DBOS, wf_id: &str) -> dbos::Result<()> {
    // Hazard 1: Resuming a workflow that is still actively executing (`PENDING`) dispatches a
    // second execution before the first notices! Always `cancel` first if it may still be live.
    let _h = dbos.resume::<(), dbos::EngineOnly>(wf_id).await?;

    // Hazard 2: `ForkOptions` does NOT inherit `queue_partition_key` from the source workflow!
    // Forking onto a partitioned queue with `queue_partition_key: None` strands the row in ENQUEUED!
    let _fork = dbos
        .fork_with::<(), dbos::EngineOnly>(
            Fork::new(wf_id, 2),
            ForkOptions {
                queue_name: Some("partitioned-orders"),
                queue_partition_key: None,
                ..ForkOptions::default()
            },
        )
        .await?;
    Ok(())
}
```

**Correct (cancelling before resume, steering forks to fixed app_versions, rewinding, and cascading):**

```rust
use dbos::{Children, DBOS, EngineOnly, Fork, ForkOptions, ResumeOptions, RewindOptions};

async fn pause_and_resume_tree(dbos: &DBOS, root_id: &str) -> dbos::Result<()> {
    // 1. Cancel root and all descendants atomically (`Children::Skip` is the default)
    dbos.cancel_all(&[root_id], Children::Include).await?;

    // 2. Resume onto a concurrency-limited queue (clears recovery_attempts and deadline)
    let _handle = dbos
        .resume_with::<String, EngineOnly>(
            root_id,
            ResumeOptions {
                queue_name: Some("recovery-throttled"),
            },
        )
        .await?;
    Ok(())
}

async fn fork_onto_patched_version(
    dbos: &DBOS,
    failed_id: &str,
    tenant_partition: &str,
) -> dbos::Result<String> {
    // Copies checkpoints `< start_step` (steps 0 and 1) into a NEW workflow_id and runs from step 2
    // on executors running `app_version = "0.2.1"`
    let forked = dbos
        .fork_with::<String, EngineOnly>(
            Fork {
                forked_id: None,
                ..Fork::new(failed_id, 2)
            },
            ForkOptions {
                app_version: Some("0.2.1"),
                queue_name: Some("partitioned-orders"),
                queue_partition_key: Some(tenant_partition),
                ..ForkOptions::default()
            },
        )
        .await?;
    Ok(forked.workflow_id().to_owned())
}

async fn rewind_settled_workflow(dbos: &DBOS, settled_id: &str) -> dbos::Result<()> {
    // Re-runs a settled (`SUCCESS`, `ERROR`, `CANCELLED`, or `MAX_RECOVERY_ATTEMPTS_EXCEEDED`)
    // workflow IN PLACE under the same `workflow_id`, discarding steps `>= 1`
    let _rewound = dbos
        .rewind_with::<String, EngineOnly>(
            settled_id,
            RewindOptions {
                start_step: 1,
                app_version: Some("0.2.1"),
                ..RewindOptions::default()
            },
        )
        .await?;
    Ok(())
}
```

### Operational Semantics Summary

- **`cancel` / `cancel_all(ids, Children)`**: Marks unsettled workflows `CANCELLED` (idempotent on terminal or nonexistent IDs). Running tasks stop cooperatively at their next step boundary (or mid-step when `StepOptions::preemptible` is `true`). A workflow cannot cancel, resume, or delete itself via `DBOS`.
- **`resume` / `resume_with` / `resume_all`**: Resets `recovery_attempts = 0` and `deadline = None` and moves the workflow to `ENQUEUED` while keeping all recorded step checkpoints intact. Fails with `NonExistentWorkflow` if the ID does not exist; no-op returning the existing outcome if already `SUCCESS` or `ERROR`.
- **`fork` / `fork_with` / `fork_all`**: Creates a **new** workflow copying step checkpoints, events, and stream entries `< start_step` (`0` re-runs from the start). Only `app_version` defaults to the source's version when `None`; `queue_name`, `queue_partition_key`, and `timeout` default to `_dbos_internal_queue`, unpartitioned, and unbounded unless set on `ForkOptions`. Use `replacement_children: &[(old_child_id, new_child_id)]` to swap child IDs in copied steps.
- **`rewind` / `rewind_with`**: Refuses active (`PENDING`, `ENQUEUED`, `DELAYED`) workflows with `WorkflowNotRewindable`. Deletes steps `>= start_step`, restores events from `workflow_events_history` `< start_step`, clears consumed notifications `>= start_step` plus unconsumed notifications, and re-enqueues the **same** `workflow_id`.
- **`delete` / `delete_all(ids, Children)`**: Permanently deletes workflow rows and cascading checkpoints/events/messages/streams. Never delete a child workflow whose parent may still replay `join_workflows` over it.

Reference: [dbos::DBOS::cancel](https://docs.rs/dbos/latest/dbos/struct.DBOS.html#method.cancel), [dbos::DBOS::resume](https://docs.rs/dbos/latest/dbos/struct.DBOS.html#method.resume), [dbos::DBOS::fork_with](https://docs.rs/dbos/latest/dbos/struct.DBOS.html#method.fork_with), [dbos::DBOS::rewind_with](https://docs.rs/dbos/latest/dbos/struct.DBOS.html#method.rewind_with)
