---
title: List and Inspect Workflows
impact: MEDIUM
impactDescription: Querying the system tables directly couples code to an internal schema
tags: workflow, list, steps, introspection, filter
---

## List and Inspect Workflows

List workflows with `list_workflows(&WorkflowFilter { .. })` (`dbos::sysdb::types::WorkflowFilter`, which implements `Default`). Filter by ID or prefix, name, status, queue, app version, time ranges, parent, attributes; set `limit`, `offset`, `sort_desc`, and `load_input`/`load_output`. `list_workflow_steps(id)` lists a workflow's recorded steps.

**Incorrect (SQL against dbos.workflow_status):**

```rust
// WRONG: the system tables are internal and change between versions.
sqlx::query("SELECT * FROM dbos.workflow_status WHERE status = 'ERROR'")
    .fetch_all(&pool)
    .await?;
```

**Correct (the listing API):**

```rust
async fn list_errors(dbos: &DBOS) -> dbos::Result<()> {
    let filter = WorkflowFilter {
        status: vec![WorkflowStatus::Error],
        limit: Some(20),
        sort_desc: true,
        ..WorkflowFilter::default()
    };
    for workflow in dbos.list_workflows(&filter).await? {
        println!(
            "{} {:?} {:?}",
            workflow.workflow_id, workflow.name, workflow.error
        );
    }
    Ok(())
}
```

```rust
async fn show_steps(dbos: &DBOS, workflow_id: &str) -> dbos::Result<()> {
    for step in dbos.list_workflow_steps(workflow_id).await? {
        println!("{} {}", step.step_id, step.step_name);
    }
    Ok(())
}
```
