---
title: Tag and Filter Workflows with Searchable Attributes
impact: MEDIUM
impactDescription: Enables structured JSONB containment queries across workflows without custom tracking tables
tags: workflow, attributes, metadata, filter, update_workflow_attributes, jsonb
---

## Tag and Filter Workflows with Searchable Attributes

Attach a `serde_json::Map<String, serde_json::Value>` via `attributes` on `RunOptions`, `StartOptions`, or `EnqueueOptions` to stamp domain metadata (tenant ID, order ID, environment, region) onto a workflow row at creation time. You can later filter workflows with PostgreSQL JSONB containment (`@>`) via `WorkflowFilter::attributes`, or replace a workflow's attributes via `DBOS::update_workflow_attributes`.

**Incorrect (passing non-object JSON filter text or assuming update_workflow_attributes merges keys):**

```rust
use dbos::DBOS;
use dbos::sysdb::types::WorkflowFilter;
use serde_json::{Map, json};

async fn bad_attribute_usage(dbos: &DBOS, workflow_id: &str) -> dbos::Result<()> {
    let mut patch = Map::new();
    patch.insert("stage".to_owned(), json!("shipped"));

    // Overwrites the ENTIRE JSON object on the row, wiping out existing `tenant_id`!
    dbos.update_workflow_attributes(workflow_id, Some(&patch)).await?;

    // Fails with InvalidArgument: `WorkflowFilter::attributes` must be JSON object text (`{...}`)
    let _rows = dbos
        .list_workflows(&WorkflowFilter {
            attributes: Some("\"shipped\""),
            ..WorkflowFilter::default()
        })
        .await?;
    Ok(())
}
```

**Correct (setting structured attributes at start, querying via JSONB containment, and replacing full maps):**

```rust
use dbos::{DBOS, StartOptions, WorkflowRef};
use dbos::sysdb::types::{WorkflowFilter, WorkflowRecord};
use serde_json::{Map, Value, json};

async fn start_tagged_order(
    fulfill: &WorkflowRef<String, String>,
    tenant_id: &str,
    order_id: &str,
) -> dbos::Result<String> {
    let mut attrs = Map::new();
    attrs.insert("tenant_id".to_owned(), json!(tenant_id));
    attrs.insert("order_id".to_owned(), json!(order_id));
    attrs.insert("tier".to_owned(), json!("enterprise"));

    let handle = fulfill
        .start_with(
            order_id.to_owned(),
            StartOptions {
                attributes: Some(&attrs),
                ..StartOptions::default()
            },
        )
        .await?;
    Ok(handle.workflow_id().to_owned())
}

async fn list_tenant_workflows(dbos: &DBOS, tenant_id: &str) -> dbos::Result<Vec<WorkflowRecord>> {
    let query = serde_json::to_string(&json!({ "tenant_id": tenant_id })).unwrap();

    // Matches any workflow whose `attributes` JSONB contains `{"tenant_id": "<tenant_id>"}`
    dbos.list_workflows(&WorkflowFilter {
        attributes: Some(&query),
        load_input: false,
        load_output: false,
        sort_desc: true,
        limit: Some(50),
        ..WorkflowFilter::default()
    })
    .await
}

async fn update_stage_preserving_existing(
    dbos: &DBOS,
    workflow_id: &str,
    stage: &str,
) -> dbos::Result<()> {
    let rows = dbos
        .list_workflows(&WorkflowFilter {
            workflow_ids: vec![workflow_id],
            load_input: false,
            load_output: false,
            ..WorkflowFilter::default()
        })
        .await?;

    let mut attrs: Map<String, Value> = rows
        .first()
        .and_then(|r| r.attributes.as_deref())
        .and_then(|raw| serde_json::from_str(raw).ok())
        .unwrap_or_default();
    attrs.insert("stage".to_owned(), json!(stage));

    // Replaces the stored JSON object (`None` clears `attributes` back to `NULL`)
    dbos.update_workflow_attributes(workflow_id, Some(&attrs)).await
}
```

### Attribute Rules and Caveats

- **Replacement, not merge**: `update_workflow_attributes(workflow_id, Some(&attrs))` replaces the entire `attributes` column; passing `None` clears `attributes` to `NULL`. Updating a missing `workflow_id` fails with `NonExistentWorkflow`.
- **Reserved `"dbos::*"` namespace**: Keys beginning with `"dbos::"` are rejected with `Error::InvalidArgument` so engine-reserved metadata cannot collide with application tags. Non-finite floats (`f64::NAN`, `f64::INFINITY`) inside attribute values are also rejected.
- **JSON object filter requirement**: `WorkflowFilter::attributes` executes a PostgreSQL JSONB `@>` containment match and must serialize a JSON object (`"{...}"`); scalars, arrays, or malformed JSON fail with `Error::InvalidArgument`.

Reference: [dbos::DBOS::update_workflow_attributes](https://docs.rs/dbos/latest/dbos/struct.DBOS.html#method.update_workflow_attributes), [dbos::sysdb::types::WorkflowFilter](https://docs.rs/dbos/latest/dbos/sysdb/types/struct.WorkflowFilter.html)
