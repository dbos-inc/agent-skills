---
title: Use Versioning for Blue-Green Deployments
impact: LOW
impactDescription: Retiring old executors before their workflows finish strands those workflows
tags: advanced, versioning, blue-green, deploy
---

## Use Versioning for Blue-Green Deployments

Give each release its own `app_version`, keep old executors running until their version's workflows finish (check with `list_workflows` filtered by `app_versions` and status), then retire them.

**Incorrect (replacing every executor at once):**

```rust
// WRONG: every deploy reuses the version and replaces every executor at once,
// so in-flight workflows replay on code that may have changed their steps.
config.app_version = Some("1.0.0".to_owned());
```

**Correct (a per-release version, and a drain check):**

```rust
fn release_config() -> Config {
    let mut config = Config::from_env("my-app");
    // A new version for every release; recovery only resumes workflows of the running version.
    config.app_version = Some(env!("CARGO_PKG_VERSION").to_owned());
    config
}

async fn old_version_drained(dbos: &DBOS, old_version: &str) -> dbos::Result<bool> {
    use dbos::sysdb::types::{WorkflowFilter, WorkflowStatus};
    let filter = WorkflowFilter {
        app_versions: vec![old_version],
        status: vec![
            WorkflowStatus::Pending,
            WorkflowStatus::Enqueued,
            WorkflowStatus::Delayed,
        ],
        limit: Some(1),
        ..WorkflowFilter::default()
    };
    Ok(dbos.list_workflows(&filter).await?.is_empty())
}
```
