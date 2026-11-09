---
title: Use Streams for Real-Time Data
impact: MEDIUM
impactDescription: Overwriting an event loses values a reader has not seen
tags: communication, streams, write_stream, read_stream
---

## Use Streams for Real-Time Data

Write an ordered sequence of values with `dbos::write_stream(key, &value)` and finish with `dbos::close_stream(key)`. Read with `dbos.read_stream::<T>(id, key, ReadStreamOptions::default())`, a `StreamReader` whose `next()` waits for each value and returns `None` when the stream is closed or the workflow has finished. Inside a workflow use `dbos::read_stream`, which records each value read.

**Incorrect (an event per value):**

```rust
async fn producer_bad(items: u32) -> dbos::Result<()> {
    for item in 0..items {
        // WRONG: an event holds one value per key, so readers miss the values
        // overwritten before they read.
        dbos::set_event("progress", &item).await?;
    }
    Ok(())
}
```

**Correct (a stream):**

```rust
#[derive(Serialize, Deserialize)]
struct Progress {
    item: u32,
    done: bool,
}

async fn process_item(item: u32) -> dbos::Result<()> {
    Ok(())
}

async fn producer_workflow(items: u32) -> dbos::Result<()> {
    for item in 0..items {
        dbos::step("process_item", || process_item(item)).await?;
        dbos::write_stream("progress", &Progress { item, done: false }).await?;
    }
    // Tell readers there's nothing more to come.
    dbos::close_stream("progress").await?;
    Ok(())
}
```

```rust
async fn watch_progress(dbos: &DBOS, workflow_id: &str) -> dbos::Result<()> {
    let mut reader =
        dbos.read_stream::<Progress>(workflow_id, "progress", ReadStreamOptions::default());
    // `next` returns `None` once the stream is closed, or the workflow has finished.
    while let Some(progress) = reader.next().await? {
        println!("Processed item {}", progress.item);
    }
    Ok(())
}
```

`read_stream_from` resumes at an offset (`reader.offset()`); `read_stream_value` reads one offset. A write from a workflow is exactly-once; from a step, at-least-once.
