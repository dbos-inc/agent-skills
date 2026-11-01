---
title: Inspect Workflow State, Steps, and Ambient Context
impact: MEDIUM
impactDescription: Avoids loading large workflow payloads unnecessarily and surfaces workflow and step IDs cleanly
tags: workflow, introspection, list_workflows, list_workflow_steps, WorkflowFilter, workflow_id, step_status
---

## Inspect Workflow State, Steps, and Ambient Context

Query workflow history through `list_workflows` and `list_workflow_steps` (available on both `DBOS` and `Client`), and read ambient execution metadata inside workflows and steps via `dbos::workflow_id()`, `dbos::step_id()`, and `dbos::step_status()`. Notice that `WorkflowFilter`, `WorkflowRecord`, `WorkflowStatus`, and `StepRecord` live under `dbos::sysdb::types` rather than the crate root.

**Incorrect (scanning unbounded workflow lists with full payloads loaded):**

```rust
use dbos::DBOS;
use dbos::sysdb::types::{WorkflowFilter, WorkflowStatus};

async fn count_failed_workflows(dbos: &DBOS) -> dbos::Result<usize> {
    // `WorkflowFilter::default()` sets `load_input: true` and `load_output: true` with no `limit`!
    // Joining and fetching megabytes of serialized inputs/outputs just to inspect status is wasteful.
    let all = dbos.list_workflows(&WorkflowFilter::default()).await?;
    Ok(all.iter().filter(|w| w.status == WorkflowStatus::Error).count())
}
```

**Correct (filtering in SQL with load_input/load_output disabled and inspecting steps/context):**

```rust
use dbos::{DBOS, ListWorkflowStepsOptions};
use dbos::sysdb::types::{StepRecord, WorkflowFilter, WorkflowRecord, WorkflowStatus};

async fn recent_failed_workflows(
    dbos: &DBOS,
    workflow_name: &str,
) -> dbos::Result<Vec<WorkflowRecord>> {
    dbos.list_workflows(&WorkflowFilter {
        status: vec![
            WorkflowStatus::Error,
            WorkflowStatus::MaxRecoveryAttemptsExceeded,
        ],
        names: vec![workflow_name],
        sort_desc: true,
        limit: Some(25),
        load_input: false,
        load_output: false,
        ..WorkflowFilter::default()
    })
    .await
}

async fn inspect_step_timeline(
    dbos: &DBOS,
    workflow_id: &str,
) -> dbos::Result<Vec<StepRecord>> {
    dbos.list_workflow_steps_with(
        workflow_id,
        ListWorkflowStepsOptions {
            load_output: false,
            limit: Some(100),
            offset: None,
        },
    )
    .await
}

async fn observable_workflow(input: String) -> dbos::Result<String> {
    // Some(id) inside a workflow or step; None outside DBOS
    let wf_id = dbos::workflow_id().expect("running inside a workflow");

    dbos::step("process_payload", || {
        let input = input.clone();
        let wf_id = wf_id.clone();
        async move {
            let status = dbos::step_status().expect("running inside a step");
            tracing::info!(
                workflow_id = %wf_id,
                step_id = status.step_id(),
                attempt = status.current_attempt(),
                "processing payload"
            );
            Ok(input)
        }
    })
    .await
}
```

### Introspection Reference

- **`WorkflowFilter` defaults (`dbos::sysdb::types::WorkflowFilter`)**: `load_input` and `load_output` both default to **`true`**, which joins `workflow_input` and `workflow_output`. Always set `load_input: false` and `load_output: false` (and set `limit`) on dashboard or monitoring queries that only need metadata.
- **Filterable dimensions**: `workflow_ids`, `workflow_id_prefixes`, `app_scope`, `status` (`Pending`, `Success`, `Error`, `MaxRecoveryAttemptsExceeded`, `Cancelled`, `Enqueued`, `Delayed`), `names`, `class_names`, `config_names`, `app_versions`, `authenticated_users`, `queues_only`, `queue_names`, `schedule_names`, `deduplication_ids`, `is_debounced`, `executor_ids`, `is_fork`, `was_forked_from`, `forked_from`, `has_parent`, `parent_workflow_ids`, `created_after` / `created_before`, `started_after` / `started_before`, `completed_after` / `completed_before` (`dbos::sysdb::types::Timestamp`), `attributes`, `sort_desc`, `limit`, and `offset`.
- **Single-workflow lookup**: Use `client.workflow_status(id).await?` (returns `Option<WorkflowStatus>`), `handle.status().await?` (returns `WorkflowStatus`), or `dbos.list_workflows(&WorkflowFilter { workflow_ids: vec![id], .. })` (there is no `DBOS::get_workflow` method).
- **Ambient context getters**:
  - `dbos::workflow_id() -> Option<String>`: `Some` in both workflow body and step body; `None` outside DBOS.
  - `dbos::step_id() -> Option<i32>`: `Some(0..)` inside a step body; `None` in the workflow body or outside DBOS.
  - `dbos::step_status() -> Option<StepStatus>`: `Some` inside a step body (`step_id()`, `current_attempt()`, `max_attempts()`); `None` everywhere else.

Reference: [dbos::DBOS::list_workflows](https://docs.rs/dbos/latest/dbos/struct.DBOS.html#method.list_workflows), [dbos::DBOS::list_workflow_steps_with](https://docs.rs/dbos/latest/dbos/struct.DBOS.html#method.list_workflow_steps_with), [dbos::workflow_id](https://docs.rs/dbos/latest/dbos/fn.workflow_id.html)
