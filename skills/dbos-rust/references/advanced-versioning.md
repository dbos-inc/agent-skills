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

**Correct (a per-release version):**

```rust
fn gpu_worker_config() -> Config {
    let mut config = Config::from_env("media-app");
    config.app_version = Some("0.1.0".to_owned());
    // Only dequeue from this queue. Some(vec![]) dequeues from none.
    config.listen_queues = Some(vec!["gpu_queue".to_owned()]);
    config
}
```
