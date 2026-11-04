---
title: Don't Capture the DBOS Instance in a Workflow
impact: HIGH
impactDescription: A captured instance is a reference cycle, so the instance and its pool are never freed
tags: workflow, ownership, leak, free-functions
---

## Don't Capture the DBOS Instance in a Workflow

The `DBOS` instance owns its registered workflow closures, so a closure that holds the instance keeps it alive forever. A workflow body never needs it: steps, sleeps, messages, events, streams and every management call have a free function in the `dbos` crate (`dbos::send`, `dbos::cancel_workflow`, `dbos::resume_workflow`, ...) that finds the running workflow on its own, and records the call as a step (except `retrieve_workflow`, which does no I/O).

**Incorrect (capturing the instance):**

```rust
fn register(dbos: &DBOS) -> dbos::Result<()> {
    let instance = dbos.clone();
    // WRONG: the closure holds the DBOS instance, which owns the closure,
    // so the instance is never freed.
    dbos.register_workflow("notify", move |order_id: String| {
        let dbos = instance.clone();
        async move {
            dbos.send(&order_id, &"shipped").await?;
            Ok::<_, dbos::Error>(())
        }
    })?;
    Ok(())
}
```

**Correct (using the free functions):**

```rust
async fn notify(order_id: String) -> dbos::Result<()> {
    // The free functions find the running workflow on their own.
    dbos::send(&order_id, &"shipped").await?;
    Ok(())
}

fn register(dbos: &DBOS) -> dbos::Result<()> {
    dbos.register_workflow("notify", notify)?;
    Ok(())
}
```

Capturing a `WorkflowRef` (to start a child) also holds the instance; for a process that launches once this is a bounded cost, and it is the way to start a child workflow today.
