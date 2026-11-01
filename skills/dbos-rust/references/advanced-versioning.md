---
title: Manage Application Versions for Rolling and Blue-Green Deploys
impact: LOW
impactDescription: Prevents replay mismatches across deployments and enables instant fleet rollbacks via Client::set_latest_app_version
tags: advanced, versioning, app_version, deployment, blue-green, rollback, Client
---

## Manage Application Versions for Rolling and Blue-Green Deploys

Every workflow row records the `app_version` it belongs to, and a `DBOS` executor **only recovers and dequeues** workflows stamped with its own `app_version` (or unversioned `Client` enqueues when this executor runs the latest registered version). Bump `Config::app_version` whenever you change workflow step sequences without `dbos::patch`, keep old executors alive until their in-flight `PENDING` / `ENQUEUED` runs drain, or promote a previous version via `Client::set_latest_app_version` to roll back.

**Incorrect (modifying workflow steps under the same app_version or shutting down old replicas immediately):**

```rust
use dbos::{Config, DBOS};

async fn unsafe_redeploy() -> dbos::Result<()> {
    // Hardcoding `"v1"` forever while adding/removing `dbos::step` calls causes v2 replicas
    // to claim in-flight `"v1"` workflows and fail with `UnexpectedStep`!
    let dbos = DBOS::new(Config {
        app_version: Some("v1".to_owned()),
        ..Config::from_env("billing-service")
    });
    dbos.launch().await?;
    Ok(())
}
```

**Correct (versioning per release, draining old versions, and promoting rollbacks via Client):**

```rust
use std::time::Duration;
use dbos::{Client, Config, DBOS, EngineOnly, Fork, ForkOptions, ShutdownOptions};
use dbos::sysdb::types::{VersionInfo, WorkflowFilter, WorkflowStatus};

async fn launch_versioned_worker(git_sha: &str) -> dbos::Result<()> {
    // Prefixing with `app_name` also keeps `version_name` globally unique on shared DBs!
    let version = format!("billing-service-{git_sha}");
    let dbos = DBOS::new(Config {
        app_version: Some(version),
        ..Config::from_env("billing-service")
    });
    let _wf = dbos.register_workflow("settle_invoice", |id: String| async move {
        Ok::<_, dbos::Error>(id)
    })?;

    dbos.launch().await?;
    println!("launched version {}", dbos.app_version()?);

    // During rolling/blue-green drain, give in-flight workflows on this version time to finish
    dbos.shutdown_with(ShutdownOptions {
        workflow_completion_timeout: Duration::from_secs(60),
    })
    .await;
    Ok(())
}

// Inspect versions, roll back unversioned enqueues, or migrate stuck runs onto a new version
async fn rollback_or_migrate_version(
    client: &Client,
    previous_good_version: &str,
    stuck_workflow_id: &str,
    fixed_version: &str,
) -> dbos::Result<Vec<VersionInfo>> {
    // 1. Inspect registered versions (ordered newest `version_timestamp` first)
    let versions = client.list_app_versions().await?;
    if let Some(latest) = client.latest_app_version().await? {
        println!("current latest version: {}", latest.version_name);
    }

    // 2. Roll back: moves `previous_good_version`'s timestamp to `now` so replicas still running
    //    it become `latest` again for unversioned `Client::enqueue` jobs and `UpdateIfLatestVersion` queues!
    client.set_latest_app_version(previous_good_version).await?;

    // 3. Migrate a failed workflow from a broken version onto `fixed_version` from step 0 (or step N)
    let _forked = client
        .fork_with::<String, EngineOnly>(
            Fork::new(stuck_workflow_id, 0),
            ForkOptions {
                app_version: Some(fixed_version),
                ..ForkOptions::default()
            },
        )
        .await?;

    // 4. Check whether a retiring old version still has unsettled workflows before scaling to 0
    let remaining = client
        .list_workflows(&WorkflowFilter {
            app_versions: vec![previous_good_version],
            status: vec![
                WorkflowStatus::Pending,
                WorkflowStatus::Enqueued,
                WorkflowStatus::Delayed,
            ],
            load_input: false,
            load_output: false,
            limit: Some(1),
            ..WorkflowFilter::default()
        })
        .await?;
    println!("unsettled workflows on {previous_good_version}: {}", remaining.len());

    Ok(versions)
}
```

### Choosing Between Blue-Green Versioning and Patching

- **Blue-green versioning (bump `app_version`)**: Zero `dbos::patch` branches in workflow code. Old replicas finish old runs while new replicas handle new runs; once `list_workflows` shows zero `PENDING`/`ENQUEUED`/`DELAYED` rows on the old version, retire the old deployment (or `fork_with` / `rewind_with` stragglers onto the new `app_version`).
- **In-place patching (keep `app_version` unchanged)**: Old replicas can shut down immediately and new replicas recover their `PENDING` rows directly, using `dbos::patch` around any added/removed/reordered steps (see [advanced-patching.md](advanced-patching.md)).

Reference: [dbos::Config::app_version](https://docs.rs/dbos/latest/dbos/struct.Config.html#structfield.app_version), [dbos::Client::set_latest_app_version](https://docs.rs/dbos/latest/dbos/struct.Client.html#method.set_latest_app_version), [dbos::sysdb::types::VersionInfo](https://docs.rs/dbos/latest/dbos/sysdb/types/struct.VersionInfo.html), [Production Checklist](https://docs.dbos.dev/production/checklist)
