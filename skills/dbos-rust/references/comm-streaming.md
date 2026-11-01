---
title: Stream Incremental Output with write_stream and read_stream
impact: MEDIUM
impactDescription: Appends ordered chunks from workflows or steps for real-time consumption without losing replay determinism
tags: comm, streaming, write_stream, close_stream, read_stream, StreamReader, sse
---

## Stream Incremental Output with write_stream and read_stream

Use `dbos::write_stream(key, &chunk).await?` and `dbos::close_stream(key).await?` to emit ordered, append-only values (LLM token chunks, progress logs, batch rows) from a workflow or step, and consume them in order with `StreamReader::next(&mut self)` obtained from `dbos::read_stream`, `dbos.read_stream`, or `client.read_stream` (plus `_from` offset variants).

**Incorrect (trying to use futures::StreamExt on StreamReader or writing streams outside a workflow):**

```rust
use dbos::{DBOS, ReadStreamOptions};

async fn bad_stream_usage(dbos: &DBOS, wf_id: &str) -> dbos::Result<()> {
    // Fails with `Error::NotInWorkflow`: only a running workflow or step can write to a stream!
    dbos::write_stream::<_, dbos::EngineOnly>("tokens", &"hello").await?;

    // Note: `StreamReader` does NOT implement `futures::Stream` because `next(&mut self)`
    // returns a durable `PendingStep<'_, Option<T>, E>`; poll it with `while let Some(v) = reader.next().await?`
    let mut reader = dbos.read_stream::<String>(wf_id, "tokens", ReadStreamOptions::default());
    let _ = reader.next().await?;
    Ok(())
}
```

**Correct (appending chunks, closing early, and consuming via StreamReader::next):**

```rust
use std::time::Duration;
use dbos::{DBOS, ReadStreamOptions};

const TOKEN_STREAM: &str = "tokens";

async fn summarize_workflow(prompt: String) -> dbos::Result<usize> {
    // Calling `write_stream` from the workflow body checkpoints each write as a step
    dbos::write_stream(TOKEN_STREAM, &format!("Starting summary for: {prompt}\n")).await?;

    // Calling `write_stream` inside `dbos::step` is ALSO allowed (appends plainly per attempt;
    // keep `max_attempts: 1` on streaming steps so retries do not duplicate chunks!)
    let chunk_count = dbos::step("stream_llm_tokens", || async {
        for token in ["Durable ", "execution ", "rocks."] {
            dbos::write_stream::<_, dbos::EngineOnly>(TOKEN_STREAM, &token.to_owned()).await?;
        }
        Ok(3_usize)
    })
    .await?;

    // Explicitly close so readers finish immediately instead of waiting for post-processing steps
    dbos::close_stream(TOKEN_STREAM).await?;

    dbos::step("archive_audit_log", || async { Ok(()) }).await?;
    Ok(chunk_count)
}

async fn consume_stream_from_handler(
    dbos: &DBOS,
    workflow_id: &str,
    resume_offset: Option<i32>,
) -> dbos::Result<Vec<String>> {
    let options = ReadStreamOptions {
        polling_interval: Some(Duration::from_millis(200)),
        timeout: Some(Duration::from_secs(30)),
    };

    let mut reader = match resume_offset {
        Some(offset) => dbos.read_stream_from::<String>(workflow_id, TOKEN_STREAM, offset, options),
        None => dbos.read_stream::<String>(workflow_id, TOKEN_STREAM, options),
    };

    let mut chunks = Vec::new();
    while let Some(chunk) = reader.next().await? {
        chunks.push(chunk);
    }
    Ok(chunks)
}
```

### Stream Semantics and Replay Guarantees

- **When a stream ends**: `reader.next().await?` yields `Ok(None)` as soon as either `dbos::close_stream(key)` is reached **or** the producer workflow enters any terminal status (`SUCCESS`, `ERROR`, `CANCELLED`, `MAX_RECOVERY_ATTEMPTS_EXCEEDED`), whichever comes first.
- **Per-value `ReadStreamOptions::timeout`**: `timeout` resets after every delivered value; if no new value arrives and the producer remains active past `timeout`, `next()` fails with `Error::StreamTimeout` (`polling_interval` must be `>= MIN_STREAM_POLLING_INTERVAL`, 1 ms).
- **Workflow readers are checkpointed per value**: When a workflow reads another workflow's stream via free `dbos::read_stream` / `dbos::read_stream_from` / `dbos::read_stream_value`, each `reader.next()` call records a `DBOS.read_stream` step so crash recovery replays the exact values seen on the first run.
- **Retries inside streaming steps**: Stream appends inside `dbos::step` are not deduplicated across retries of that step; use `max_attempts: 1` on steps that stream partial tokens, or emit chunks from the workflow body between steps.

Reference: [dbos::write_stream](https://docs.rs/dbos/latest/dbos/fn.write_stream.html), [dbos::close_stream](https://docs.rs/dbos/latest/dbos/fn.close_stream.html), [dbos::StreamReader](https://docs.rs/dbos/latest/dbos/struct.StreamReader.html), [dbos::ReadStreamOptions](https://docs.rs/dbos/latest/dbos/struct.ReadStreamOptions.html)
