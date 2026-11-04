---
title: Manage Application Versions for Safe Code Upgrades
impact: MEDIUM
impactDescription: Recovery only resumes workflows started on the running version
tags: advanced, versioning, app_version, recovery
---

## Manage Application Versions for Safe Code Upgrades

Every workflow is tagged with the application version it started on, and recovery resumes only workflows of the running version. DBOS for Rust doesn't compute a version: set `config.app_version` (or `DBOS__APPVERSION`). Pin one value across deployments to use patching, or change it per release for blue-green deploys. `env!("CARGO_PKG_VERSION")` or a commit SHA are typical choices.

**Incorrect (no version):**

```rust
async fn my_workflow(_: ()) -> dbos::Result<String> {
    Ok("done".to_owned())
}

#[tokio::main]
async fn main() -> dbos::Result<()> {
    // No app_version: launch fails unless DBOS__APPVERSION is set.
    let dbos = DBOS::new(Config::from_env("my-app"));
    dbos.launch().await?;
    // Registered after launch: refused with Error::AlreadyLaunched.
    let workflow = dbos.register_workflow("my_workflow", my_workflow)?;
    workflow.run(()).await?;
    Ok(())
}
```

**Correct (a pinned version):**

```rust
fn config() -> dbos::Config {
    let mut config = dbos::Config::from_env("my-app");
    // Every deployment shares this version, so new code recovers old workflows.
    config.app_version = Some("1.0.0".to_owned());
    config
}
```

An unversioned enqueue (a client's default) is dequeued only by the application's latest version. A workflow stranded on a retired version can be moved with `fork_with`/`rewind_with` and `app_version`.
