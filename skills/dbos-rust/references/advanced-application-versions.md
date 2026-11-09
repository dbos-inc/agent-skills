---
title: Manage Application Versions for Safe Code Upgrades
impact: MEDIUM
impactDescription: Recovery only resumes workflows started on the running version
tags: advanced, versioning, app_version, recovery
---

## Manage Application Versions for Safe Code Upgrades

Every workflow is tagged with the application version it started on, and recovery resumes only workflows of the running version. DBOS for Rust doesn't compute a version: set `config.app_version` (or `DBOS__APPVERSION`). Pin one value across deployments to use patching, or change it per release for blue-green deploys. `env!("CARGO_PKG_VERSION")` or a commit SHA are typical choices.

**Incorrect (a version that changes on every build):**

```rust
// WRONG: a new version on every build, even when no workflow changed, strands the
// previous build's PENDING workflows: no executor of their version is left to recover them.
config.app_version = Some(format!("{}", std::time::SystemTime::now().duration_since(std::time::UNIX_EPOCH)?.as_secs()));
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
