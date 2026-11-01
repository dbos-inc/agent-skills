---
title: Run Recurring Cron Jobs with Database-Backed Schedules
impact: MEDIUM
impactDescription: Reconciles cron schedules in PostgreSQL so every tick runs once across a multi-replica fleet
tags: pattern, schedule, cron, ScheduleSpec, ScheduledWorkflowInput, apply_schedules, backfill
---

## Run Recurring Cron Jobs with Database-Backed Schedules

In DBOS Rust, schedules are rows in `dbos.workflow_schedules` rather than code decorators. Register a workflow whose parameter is `ScheduledWorkflowInput<C>`, launch DBOS, and declare schedules via `dbos.apply_schedules(&specs).await?` (or `create_schedule`). Every replica runs the scheduler loop and derives a deterministic workflow ID per `(schedule_name, tick)`, so the fleet executes each cron tick **at most once** regardless of replica count.

**Incorrect (writing an in-memory loop with tokio::time::interval or wrong workflow input type):**

```rust
use std::time::Duration;
use dbos::WorkflowRef;

async fn wrong_cron_loop(report_wf: WorkflowRef<(), ()>) {
    // Runs on EVERY replica simultaneously (N duplicates per tick!) and loses missed ticks on deploys!
    let mut ticker = tokio::time::interval(Duration::from_secs(3600));
    loop {
        ticker.tick().await;
        let _ = report_wf.start(()).await;
    }
}
```

**Correct (registering ScheduledWorkflowInput<C> and reconciling with apply_schedules):**

```rust
use std::time::SystemTime;
use dbos::{
    Config, DBOS, EngineOnly, QueueConflict, QueueOptions, ScheduleSpec, ScheduledWorkflowInput,
};
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
struct ReportConfig {
    region: String,
    include_archived: bool,
}

async fn hourly_report(input: ScheduledWorkflowInput<ReportConfig>) -> dbos::Result<String> {
    // `input.scheduled_time` is the deterministic cron tick (`SystemTime`), safe to read on replay!
    let tick_epoch = input
        .scheduled_time
        .duration_since(SystemTime::UNIX_EPOCH)
        .unwrap_or_default()
        .as_secs();
    let region = input.context.region;

    dbos::step("generate_report", || {
        let region = region.clone();
        async move { Ok(format!("{region}@{tick_epoch}")) }
    })
    .await
}

async fn setup_schedules() -> dbos::Result<()> {
    let dbos = DBOS::new(Config {
        app_version: Some("0.1.0".to_owned()),
        ..Config::from_env("reporting-app")
    });
    let report_wf = dbos.register_workflow("hourly_report", hourly_report)?;

    dbos.launch().await?;
    dbos.register_queue("reports", QueueOptions::default(), QueueConflict::NeverUpdate)
        .await?;

    let mut eu_spec = ScheduleSpec::new(
        "reporting-app-eu-hourly",
        &report_wf,
        "0 0 * * * *",
        &ReportConfig {
            region: "eu-west".to_owned(),
            include_archived: false,
        },
    )?;
    eu_spec.cron_timezone = Some("Europe/Berlin".to_owned());
    eu_spec.queue_name = Some("reports".to_owned());
    eu_spec.automatic_backfill = true;

    // Upserts every spec in the slice (preserving existing `PAUSED` status if already paused);
    // does NOT delete other schedules omitted from the slice
    dbos.apply_schedules(&[eu_spec]).await?;

    // Runtime controls (no restart needed): pause, resume, trigger now, or backfill a window
    dbos.pause_schedule("reporting-app-eu-hourly").await?;
    dbos.resume_schedule("reporting-app-eu-hourly").await?;
    let _now_handle = dbos
        .trigger_schedule::<String, EngineOnly>("reporting-app-eu-hourly")
        .await?;

    Ok(())
}
```

### Cron Dialect and Schedule Rules

DBOS Rust accepts 5-field (minute resolution, firing at second `0`) or 6-field (seconds first) cron expressions, plus nicknames (`@hourly`, `@daily`, `@midnight`, `@weekly`, `@monthly`, `@yearly`, `@annually`):

```text
# 6-field (with seconds):
# ┌───────────── second (0-59)
# │ ┌─────────── minute (0-59)
# │ │ ┌───────── hour (0-23)
# │ │ │ ┌─────── day of month (1-31, L, LW, nW, ?)
# │ │ │ │ ┌───── month (1-12 or JAN-DEC)
# │ │ │ │ │ ┌─── day of week (0-7 or SUN-SAT, nL, n#m, ?)
# * * * * * *
```

- **Global `schedule_name` uniqueness**: `workflow_schedules.schedule_name` is unique across the entire system database schema, so prefix schedule names with `app_name` when sharing a database across services.
- **Custom queue must exist first**: If `ScheduleSpec::queue_name` is `Some("reports")`, register `"reports"` via `register_queue` **before** calling `create_schedule` or `apply_schedules` (`None` uses `_dbos_internal_queue`).
- **Workflow-context restrictions**: Single-schedule CRUD (`create_schedule`, `update_schedule`, `list_schedules`, `get_schedule`, `pause_schedule`, `resume_schedule`, `delete_schedule`) is checkpointed when called on `DBOS` inside a workflow; `apply_schedules`, `backfill_schedule`, and `trigger_schedule` are refused inside workflows (`Error::InsideWorkflow`).

Reference: [dbos::ScheduleSpec](https://docs.rs/dbos/latest/dbos/struct.ScheduleSpec.html), [dbos::ScheduledWorkflowInput](https://docs.rs/dbos/latest/dbos/struct.ScheduledWorkflowInput.html), [dbos::DBOS::apply_schedules](https://docs.rs/dbos/latest/dbos/struct.DBOS.html#method.apply_schedules)
