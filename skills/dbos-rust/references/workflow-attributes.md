---
title: Tag Workflows with Searchable Attributes
impact: MEDIUM
impactDescription: Without attributes, finding workflows for one customer or region means scanning them all
tags: workflow, attributes, search, list
---

## Tag Workflows with Searchable Attributes

Attach JSON key-value attributes with the `attributes` field of `StartOptions` or `EnqueueOptions`, and find workflows by them with the `attributes` field of `WorkflowFilter`, which matches workflows whose attributes contain the given JSON.

**Incorrect (encoding metadata in the workflow ID):**

```rust
// WRONG: metadata packed into the ID can only be found by prefix.
let id = format!("acme-eu-{order_id}");
```

**Correct (attributes at start, filter on them):**

```rust
async fn tag_order(process_order: &WorkflowRef<String, String>) -> dbos::Result<()> {
    let mut attributes = serde_json::Map::new();
    attributes.insert("customer".to_owned(), "acme".into());
    attributes.insert("region".to_owned(), "eu".into());
    let options = StartOptions {
        attributes: Some(&attributes),
        ..StartOptions::default()
    };
    process_order
        .start_with("order-42".to_owned(), options)
        .await?;
    Ok(())
}

async fn find_acme_orders(dbos: &DBOS) -> dbos::Result<()> {
    let filter = WorkflowFilter {
        // The workflow's attributes must contain this JSON.
        attributes: Some(r#"{"customer": "acme"}"#),
        ..WorkflowFilter::default()
    };
    let workflows = dbos.list_workflows(&filter).await?;
    Ok(())
}
```

Replace a workflow's attributes with `update_workflow_attributes`, or merge keys in with `merge_workflow_attributes`.
