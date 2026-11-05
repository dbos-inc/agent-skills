---
title: Initialize Client for External Access
impact: HIGH
impactDescription: A separate service that launches DBOS becomes an executor that runs other services' work
tags: client, external, setup
---

## Initialize Client for External Access

A service that only enqueues, sends, reads events or manages workflows connects with `Client::connect(ClientConfig::from_env())`, not a launched `DBOS`. A client runs no workflows. Set `ClientConfig::app_name` when several applications share the system database: named, the client writes and reads that application's rows; `None` writes unclaimed rows and reads all applications'.

**Incorrect (launching DBOS in a service that runs no workflows):**

```rust
async fn status_bad(dbos: &DBOS) {
    // WRONG: a separate service doesn't need a DBOS instance of its own;
    // launching one would make it an executor that recovers and dequeues work.
    dbos.launch().await.unwrap();
}
```

**Correct (a client):**

```rust
async fn client_example() -> dbos::Result<()> {
    let client = Client::connect(ClientConfig {
        // Act for this application when several share the database.
        app_name: Some("order-service".to_owned()),
        ..ClientConfig::from_env()
    })
    .await?;
    // ... enqueue, send, get_event, list_workflows, ...
    client.close().await;
    Ok(())
}
```

`ClientConfig` defaults: `max_connections` 5, `use_listen_notify` false (turn it on only if the application's executors use LISTEN/NOTIFY), `schema` `"dbos"`, which must match the application's.
