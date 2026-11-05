---
title: Control Which Queues a Worker Listens To
impact: HIGH
impactDescription: Workers dequeue work they are not provisioned for
tags: queue, listen, worker, config
---

## Control Which Queues a Worker Listens To

By default a process dequeues from every queue its application owns. Set `config.listen_queues = Some(vec![..])` to limit it, or `Some(vec![])` for a process that enqueues but never runs queued work. DBOS's internal queue (resumed and forked workflows) is always dequeued.

**Incorrect (one config for every machine):**

```rust
fn gpu_worker_config_bad() -> Config {
    // WRONG: this process dequeues from every queue, so a CPU-only machine
    // can pick up GPU work, and the reverse.
    Config::from_env("media-app")
}
```

**Correct (a GPU worker):**

```rust
fn gpu_worker_config() -> Config {
    let mut config = Config::from_env("media-app");
    config.app_version = Some("0.1.0".to_owned());
    // Only dequeue from this queue. Some(vec![]) dequeues from none.
    config.listen_queues = Some(vec!["gpu_queue".to_owned()]);
    config
}
```
