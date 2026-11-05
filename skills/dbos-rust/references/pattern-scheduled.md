---
title: Create Scheduled Workflows
impact: MEDIUM
impactDescription: A hand-rolled timer loop misses ticks across restarts and runs them twice across processes
tags: pattern, schedule, cron, backfill
---

## Create Scheduled Workflows

A scheduled workflow takes a `ScheduledWorkflowInput<C>` (`scheduled_time`, `context: C`). After `launch`, create a schedule with `ScheduleSpec::new(name, &workflow_ref, cron, &context)` and `create_schedule`, or create-or-replace a set with `apply_schedules`. Cron expressions have five fields, or six with seconds first; `cron_timezone` sets an IANA zone (UTC default). Each tick runs once across all executors.

**Incorrect (a loop with tokio::time::interval):**

```rust
// WRONG: every process runs every tick, and ticks during downtime are lost.
tokio::spawn(async move {
    let mut tick = tokio::time::interval(Duration::from_secs(24 * 60 * 60));
    loop {
        tick.tick().await;
        report.start(()).await.ok();
    }
});
```

**Correct (a durable schedule):**

```rust
async fn create(dbos: &DBOS) -> dbos::Result<()> {
    // Registered before launch:
    let report = dbos.register_workflow("nightly_report", nightly_report)?;
    dbos.launch().await?;

    // Every day at 2am.
    let context = ReportContext {
        region: "emea".to_owned(),
    };
    let mut spec = ScheduleSpec::new("nightly-emea", &report, "0 2 * * *", &context)?;
    spec.cron_timezone = Some("Europe/Paris".to_owned());
    dbos.create_schedule(&spec).await?;
    Ok(())
}
```

```rust
#[derive(Serialize, Deserialize)]
struct ReportContext {
    region: String,
}

async fn nightly_report(input: ScheduledWorkflowInput<ReportContext>) -> dbos::Result<()> {
    println!(
        "Report for {} at {:?}",
        input.context.region, input.scheduled_time
    );
    Ok(())
}
```

Manage schedules with `pause_schedule`, `resume_schedule`, `update_schedule` (`ScheduleChange`), `list_schedules`, `get_schedule`, `delete_schedule`; run missed ticks with `backfill_schedule`, or one now with `trigger_schedule`. `apply_schedules`, `backfill_schedule` and `trigger_schedule` can't be called from a workflow. Set `automatic_backfill` to run missed ticks on startup, and `queue_name` to run ticks on a registered queue.
