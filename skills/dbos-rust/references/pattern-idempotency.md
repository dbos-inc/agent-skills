---
title: Use Workflow IDs for Idempotency
impact: MEDIUM
impactDescription: Retried requests repeat payments, emails and other side effects
tags: pattern, idempotency, workflow-id
---

## Use Workflow IDs for Idempotency

A workflow ID is an idempotency key: starting a workflow with an ID that already exists returns the existing workflow (`WorkflowIdReusePolicy::ReturnExisting`, the default) instead of running it again. Derive the ID from the request (an order or payment ID) and set it in `StartOptions::workflow_id`. Read the running workflow's ID with `dbos::workflow_id()`.

**Incorrect (a fresh ID for every retry):**

```rust
// WRONG: every retry of the request starts another payment.
let handle = process_payment.start(payment).await?;
```

**Correct (the request's ID as the workflow ID):**

```rust
async fn example_workflow(input: String) -> dbos::Result<String> {
    let id = dbos::workflow_id().expect("called inside a workflow");
    println!("Running workflow with ID: {id}");
    Ok("success".to_owned())
}

async fn workflow_id_example(example_workflow: &WorkflowRef<String, String>) -> dbos::Result<()> {
    let options = StartOptions {
        workflow_id: Some("unique-workflow-id-123"),
        ..StartOptions::default()
    };
    let handle = example_workflow
        .start_with("input".to_owned(), options)
        .await?;
    println!("Result: {}", handle.result().await?);
    Ok(())
}
```

To fail instead of joining, set `workflow_id_reuse_policy: WorkflowIdReusePolicy::Reject`:

```rust
async fn reject_example(example_workflow: &WorkflowRef<String, String>) -> dbos::Result<()> {
    let options = StartOptions {
        workflow_id: Some("very-unique-id"),
        workflow_id_reuse_policy: WorkflowIdReusePolicy::Reject,
        ..StartOptions::default()
    };
    match example_workflow
        .start_with("input".to_owned(), options)
        .await
    {
        Ok(handle) => println!("started {}", handle.workflow_id()),
        Err(dbos::Error::SystemDatabase(dbos::sysdb::Error::WorkflowIdInUse { .. })) => {
            // A workflow with this ID already exists.
        }
        Err(error) => return Err(error),
    }
    Ok(())
}
```
