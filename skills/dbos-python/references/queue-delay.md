---
title: Delay Enqueued Workflows
impact: MEDIUM
impactDescription: Schedule workflows to run at a future time
tags: queue, delay, delayed, scheduling, set_workflow_delay
---

## Delay Enqueued Workflows

Use `delay_seconds` in `SetEnqueueOptions` to schedule a workflow to run after a delay. The workflow is placed in `DELAYED` status and does not execute until the delay expires.

**Incorrect (sleeping or scheduling in app code to defer work):**

```python
import time
import threading

def schedule_reminder(user_id: str):
    # Loses the schedule on restart!
    def fire():
        time.sleep(3600)
        DBOS.start_workflow(send_reminder, user_id)
    threading.Thread(target=fire).start()
```

**Correct (delayed enqueue):**

```python
from dbos import DBOS, SetEnqueueOptions

DBOS.register_queue("reminders")

@DBOS.workflow()
def send_reminder(user_id: str):
    ...

# Send a reminder in one hour. The delay is durable -
# it survives restarts.
with SetEnqueueOptions(delay_seconds=3600):
    handle = DBOS.enqueue_workflow("reminders", send_reminder, user_id)
```

After the delay expires, the workflow transitions from `DELAYED` to `ENQUEUED` and is dequeued normally.

### Updating a Delay Dynamically

Use `DBOS.set_workflow_delay` to change the delay on a workflow that is still in `DELAYED` status. Provide exactly one of `delay_seconds` (relative) or `delay_until_epoch_ms` (absolute).

```python
import time

# Shorten to fire 10 seconds from now
DBOS.set_workflow_delay(handle.workflow_id, delay_seconds=10)

# Or pin to an absolute deadline
DBOS.set_workflow_delay(
    handle.workflow_id,
    delay_until_epoch_ms=int((time.time() + 60) * 1000),
)
```

Only affects workflows still in `DELAYED` status — once the workflow has transitioned to `ENQUEUED` or beyond, the call is a no-op.

Available on `DBOSClient` as `client.set_workflow_delay` for external services.

Use cases:
- Scheduled reminders / notifications
- Retry-after-N-seconds patterns
- Cool-down periods before re-running a workflow

Reference: [Delayed Execution](https://docs.dbos.dev/python/tutorials/queue-tutorial#delayed-execution)
