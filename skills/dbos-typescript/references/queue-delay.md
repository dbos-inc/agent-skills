---
title: Delay Enqueued Workflows
impact: MEDIUM
impactDescription: Schedule workflows to run at a future time
tags: queue, delay, delayed, scheduling, setWorkflowDelay
---

## Delay Enqueued Workflows

Use `delaySeconds` in `enqueueOptions` to schedule a workflow to run after a delay. The workflow is placed in `DELAYED` status and does not execute until the delay expires.

**Incorrect (using setTimeout / app-level scheduling):**

```typescript
function scheduleReminder(userId: string) {
  // Loses the schedule if the process restarts!
  setTimeout(async () => {
    await DBOS.startWorkflow(sendReminder)(userId);
  }, 3600 * 1000);
}
```

**Correct (delayed enqueue):**

```typescript
async function sendReminderFn(userId: string) {
  // ...
}
const sendReminder = DBOS.registerWorkflow(sendReminderFn, { name: "sendReminder" });

await DBOS.registerQueue("reminders");

// Send a reminder in one hour. The delay is durable -
// it survives restarts.
const handle = await DBOS.startWorkflow(sendReminder, {
  queueName: "reminders",
  enqueueOptions: { delaySeconds: 3600 },
})(userId);
```

After the delay expires, the workflow transitions from `DELAYED` to `ENQUEUED` and is dequeued normally.

### Updating a Delay Dynamically

Use `DBOS.setWorkflowDelay` to change the delay on a workflow that is still in `DELAYED` status. Provide exactly one of `delaySeconds` (relative) or `delayUntilEpochMS` (absolute).

```typescript
// Shorten to fire 10 seconds from now
await DBOS.setWorkflowDelay(handle.workflowID, { delaySeconds: 10 });

// Or pin to an absolute deadline
await DBOS.setWorkflowDelay(handle.workflowID, {
  delayUntilEpochMS: Date.now() + 60000,
});
```

Only affects workflows still in `DELAYED` status — once the workflow has transitioned to `ENQUEUED` or beyond, the call is a no-op.

Use cases:
- Scheduled reminders / notifications
- Retry-after-N-seconds patterns
- Cool-down periods before re-running a workflow

Reference: [Delayed Execution](https://docs.dbos.dev/typescript/tutorials/queue-tutorial#delayed-execution)
