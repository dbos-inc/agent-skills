---
title: Route Work to Dedicated Worker Pools with Config::listen_queues
impact: MEDIUM
impactDescription: Prevents API nodes or CPU-light replicas from claiming GPU or memory-heavy queue jobs
tags: queue, listen_queues, Config, worker-pool, topology
---

## Route Work to Dedicated Worker Pools with Config::listen_queues

By default (`Config::listen_queues: None`), **every** launched `DBOS` replica of an application discovers and dequeues from **every** queue registered under that `app_name` in PostgreSQL (even queues registered only by another replica!). Set `Config::listen_queues` before `launch()` to isolate heterogeneous workloads onto dedicated worker pools or to make HTTP frontend replicas enqueue-only.

**Incorrect (assuming a replica only polls queues it called register_queue on):**

```rust
use dbos::{Config, DBOS, QueueConflict, QueueOptions};

async fn accidental_gpu_claim_on_api_node() -> dbos::Result<()> {
    // `listen_queues` defaults to `None` (poll ALL queues owned by `"media-app"`)!
    // Even though this API replica never calls `register_queue("gpu-transcode", ..)`,
    // its dequeue supervisor reads `dbos.queues` and will claim GPU jobs and run them here!
    let api_dbos = DBOS::new(Config {
        app_version: Some("0.1.0".to_owned()),
        ..Config::from_env("media-app")
    });
    let _transcode = api_dbos.register_workflow("transcode", |url: String| async move {
        Ok::<_, dbos::Error>(url)
    })?;
    api_dbos.launch().await?;

    let _ = QueueOptions::default();
    let _ = QueueConflict::NeverUpdate;
    Ok(())
}
```

**Correct (scoping API nodes to Some(vec![]) and GPU workers to Some(vec!["gpu-transcode"])):**

```rust
use dbos::{Config, DBOS, Enqueue, QueueConflict, QueueOptions, StartOptions};

async fn launch_enqueue_only_api_node() -> dbos::Result<()> {
    // `Some(vec![])` polls NO user queues (still serves HTTP handlers, direct `run`/`start`,
    // and `_dbos_internal_queue` for `resume`/`fork`/schedules)
    let api_dbos = DBOS::new(Config {
        app_version: Some("0.1.0".to_owned()),
        listen_queues: Some(vec![]),
        ..Config::from_env("media-app")
    });
    let transcode = api_dbos.register_workflow("transcode", |url: String| async move {
        Ok::<_, dbos::Error>(url)
    })?;
    api_dbos.launch().await?;

    api_dbos
        .register_queue(
            "gpu-transcode",
            QueueOptions {
                worker_concurrency: Some(2),
                ..QueueOptions::default()
            },
            QueueConflict::UpdateIfLatestVersion,
        )
        .await?;

    // Enqueues onto `"gpu-transcode"` for the GPU worker fleet to claim
    transcode
        .start_with(
            "s3://clips/in.mp4".to_owned(),
            StartOptions {
                queue: Some(Enqueue::new("gpu-transcode")),
                ..StartOptions::default()
            },
        )
        .await?;
    Ok(())
}

async fn launch_dedicated_gpu_worker() -> dbos::Result<()> {
    // Dequeues ONLY `"gpu-transcode"` (plus `_dbos_internal_queue`)
    let gpu_dbos = DBOS::new(Config {
        app_version: Some("0.1.0".to_owned()),
        listen_queues: Some(vec!["gpu-transcode".to_owned()]),
        ..Config::from_env("media-app")
    });
    let _transcode = gpu_dbos.register_workflow("transcode", |url: String| async move {
        Ok::<_, dbos::Error>(url)
    })?;
    gpu_dbos.launch().await?;
    Ok(())
}
```

### `listen_queues` Rules

- **Three states of `Option<Vec<String>>`**:
  - `None` *(default)*: Dequeue from all queues belonging to this application (plus unclaimed queues).
  - `Some(vec!["q1".into(), "q2".into()])`: Dequeue only from `"q1"` and `"q2"`.
  - `Some(vec![])`: Dequeue from **zero** user queues. (If the process never runs workflows directly or hosts schedules either, consider using `dbos::Client` instead; see [client-setup.md](client-setup.md).)
- **`_dbos_internal_queue` is always polled**: Every launched `DBOS` executor always listens on `"_dbos_internal_queue"` so `resume`, `fork`, `rewind`, and default cron schedules continue to run; passing `""` or `"_dbos_internal_queue"` inside `listen_queues` fails validation at `launch()` with `Error::Config`.

Reference: [dbos::Config::listen_queues](https://docs.rs/dbos/latest/dbos/struct.Config.html#structfield.listen_queues)
