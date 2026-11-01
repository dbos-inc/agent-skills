---
title: Register Database-Backed Queues After Launch and Enqueue by Name
impact: HIGH
impactDescription: Decouples workflow submission from bounded execution across your worker fleet
tags: queue, register_queue, Enqueue, QueueOptions, QueueConflict, background
---

## Register Database-Backed Queues After Launch and Enqueue by Name

Every queue in DBOS Rust is a row in the `dbos.queues` table rather than an in-memory object. Register queues **after** `dbos.launch().await?` using `dbos.register_queue(name, options, conflict).await?` (unlike workflows, which are registered before `launch()`), and enqueue workflows onto the queue by name using `Enqueue::new("queue-name")` on `StartOptions` or `RunOptions`.

**Incorrect (calling register_queue before launch or spawning unbounded background workflows):**

```rust
use dbos::{Config, DBOS, QueueConflict, QueueOptions};

async fn bad_queue_setup(items: Vec<String>) -> dbos::Result<()> {
    let dbos = DBOS::new(Config {
        app_version: Some("0.1.0".to_owned()),
        ..Config::from_env("batch-worker")
    });
    let process = dbos.register_workflow("process_item", |item: String| async move {
        Ok::<_, dbos::Error>(item)
    })?;

    // Fails with `Error::NotLaunched`! A queue is a database row and requires a live connection
    dbos.register_queue("items", QueueOptions::default(), QueueConflict::NeverUpdate)
        .await?;

    dbos.launch().await?;

    // Starts all 10,000 workflows concurrently in this single process with no backpressure!
    for item in items {
        process.start(item).await?;
    }
    Ok(())
}
```

**Correct (registering queue after launch and submitting via StartOptions::queue):**

```rust
use dbos::{
    Config, DBOS, Enqueue, QueueConflict, QueueOptions, StartOptions, WorkflowHandle,
};

async fn enqueue_batch(items: Vec<String>) -> dbos::Result<Vec<String>> {
    let dbos = DBOS::new(Config {
        app_version: Some("0.1.0".to_owned()),
        ..Config::from_env("batch-worker")
    });
    let process = dbos.register_workflow("process_item", |item: String| async move {
        Ok::<_, dbos::Error>(format!("done:{item}"))
    })?;

    // 1. Launch DBOS first
    dbos.launch().await?;

    // 2. Seed or update the database-backed queue row
    dbos.register_queue(
        "items",
        QueueOptions {
            worker_concurrency: Some(10),
            concurrency: Some(50),
            ..QueueOptions::default()
        },
        QueueConflict::UpdateIfLatestVersion,
    )
    .await?;

    // 3. Enqueue workflows onto `"items"`; workers across the fleet claim them in FIFO order
    let mut handles: Vec<WorkflowHandle<String>> = Vec::with_capacity(items.len());
    for item in items {
        let handle = process
            .start_with(
                item,
                StartOptions {
                    queue: Some(Enqueue::new("items")),
                    ..StartOptions::default()
                },
            )
            .await?;
        handles.push(handle);
    }

    let mut outputs = Vec::with_capacity(handles.len());
    for handle in handles {
        outputs.push(handle.result().await?);
    }

    dbos.shutdown().await;
    Ok(outputs)
}
```

### Choosing a `QueueConflict` Strategy

`QueueConflict` has no `Default` so every `register_queue` call explicitly declares how rolling deploys and runtime edits interact:

- `QueueConflict::UpdateIfLatestVersion`: Updates the stored queue row only if this process runs the newest registered `app_version` (skipping overwrites from older replicas still draining during a rolling deploy). Recommended when code owns queue limits.
- `QueueConflict::NeverUpdate`: Inserts the queue if absent and leaves an existing row untouched. Recommended when operators tune queue limits live via `update_queue` and restarts must not revert those changes.
- `QueueConflict::AlwaysUpdate`: Unconditionally overwrites the stored limits (convenient in `Client` admin scripts and isolated tests).

Note that the reserved internal queue (`"_dbos_internal_queue"`) is managed automatically by DBOS and rejected by `register_queue`.

Reference: [dbos::DBOS::register_queue](https://docs.rs/dbos/latest/dbos/struct.DBOS.html#method.register_queue), [dbos::QueueOptions](https://docs.rs/dbos/latest/dbos/struct.QueueOptions.html), [dbos::Enqueue](https://docs.rs/dbos/latest/dbos/struct.Enqueue.html)
