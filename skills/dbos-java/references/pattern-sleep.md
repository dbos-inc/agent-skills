---
title: Use Durable Sleep Inside Workflows
impact: MEDIUM
impactDescription: Wake-up times survive restarts, enabling waits of days or weeks
tags: pattern, sleep, durable, timing, delay
---

## Use Durable Sleep Inside Workflows

`dbos.sleep(Duration)` records the intended wake-up time in the database. If the process restarts during the sleep,
the workflow still wakes at the original time instead of sleeping the full duration again. `Thread.sleep` has no
such guarantee and blocks a thread for the whole wait.

**Incorrect (Thread.sleep in a workflow):**

```java
@Workflow
public void reminderWorkflow(String userId) throws InterruptedException {
  Thread.sleep(Duration.ofDays(7).toMillis()); // restarts the full week on recovery
  dbos.runStep(() -> sendReminder(userId), "sendReminder");
}
```

**Correct (durable sleep):**

```java
@Workflow
public void reminderWorkflow(String userId) {
  dbos.sleep(Duration.ofDays(7)); // wake-up time is checkpointed
  dbos.runStep(() -> sendReminder(userId), "sendReminder");
}
```

Behavior:

- Inside a workflow, the sleep is durable and each `sleep` call is a checkpoint
- Called from a step or from outside a workflow, `dbos.sleep` behaves like an ordinary `Thread.sleep`
- Sleeping workflows consume no thread while waiting, so waits of days, weeks, or months are practical
- A cancelled or timed-out workflow is preempted at the next step boundary rather than mid-sleep

Choosing between mechanisms:

- Wait inside a running workflow: `dbos.sleep`
- Postpone a workflow that has not started yet: `StartWorkflowOptions.withDelay`
  ([queue-delay.md](queue-delay.md))
- Recurring work: schedules ([pattern-scheduled.md](pattern-scheduled.md))

Reference: [Durable Sleep](https://docs.dbos.dev/java/tutorials/workflow-tutorial#durable-sleep)
