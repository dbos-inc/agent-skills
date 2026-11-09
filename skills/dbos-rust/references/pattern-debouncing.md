---
title: Debounce Workflows to Prevent Wasted Work
impact: MEDIUM
impactDescription: Rapid triggers each start a workflow, wasting work on superseded input
tags: pattern, debounce, coalesce
---

## Debounce Workflows to Prevent Wasted Work

A `Debouncer` delays a workflow until `period` has passed since the last `debounce(key, period, input)` for that key, then runs it once with the latest input. `debounce_timeout` caps the total delay from the first call. Different keys debounce independently. Every call that coalesces returns a handle to the same workflow.

**Incorrect (starting a workflow on every keystroke):**

```rust
// WRONG: a workflow per keystroke, each with stale input.
async fn on_edit(process_input: &WorkflowRef<String, String>, text: String) -> dbos::Result<()> {
    process_input.start(text).await?;
    Ok(())
}
```

**Correct (a debouncer):**

```rust
async fn process_input(input: String) -> dbos::Result<String> {
    Ok(input)
}

async fn debounce_example(dbos: &DBOS) -> dbos::Result<()> {
    // Register workflows before launch, and debounce after it.
    let process_input = dbos.register_workflow("process_input", process_input)?;
    dbos.launch().await?;

    let debouncer = dbos::Debouncer {
        // Never hold the workflow back more than a minute after the first call.
        debounce_timeout: Some(Duration::from_secs(60)),
        ..dbos::Debouncer::new(&process_input)
    };
    // Each call within five seconds of the last pushes the start back;
    // the workflow runs once, with the last input.
    let handle = debouncer
        .debounce(
            "user-123",
            Duration::from_secs(5),
            "latest input".to_owned(),
        )
        .await?;
    Ok(())
}
```

From another application, use `DebouncerClient::new(&client, workflow_name, queue)`.
