---
title: Use Workflow IDs for Idempotency
impact: MEDIUM
impactDescription: Prevents duplicate executions of critical operations
tags: idempotency, workflow-id, deduplication, exactly-once, workflow_id_reuse_policy
---

## Use Workflow IDs for Idempotency

Set workflow IDs to make operations idempotent. A workflow with the same ID executes only once.

**Incorrect (duplicate payments possible):**

```python
@app.post("/pay/{order_id}")
def process_payment(order_id: str):
    # Multiple clicks = multiple payments!
    handle = DBOS.start_workflow(payment_workflow, order_id)
    return handle.get_result()
```

**Correct (idempotent with workflow ID):**

```python
from dbos import SetWorkflowID

@app.post("/pay/{order_id}")
def process_payment(order_id: str):
    # Same order_id = same workflow ID = only one execution
    with SetWorkflowID(f"payment-{order_id}"):
        handle = DBOS.start_workflow(payment_workflow, order_id)
    return handle.get_result()

@DBOS.workflow()
def payment_workflow(order_id: str):
    charge_customer(order_id)
    send_confirmation(order_id)
    return "success"
```

Access the workflow ID inside workflows:

```python
@DBOS.workflow()
def my_workflow():
    current_id = DBOS.workflow_id
    DBOS.logger.info(f"Running workflow {current_id}")
```

Workflow IDs must be globally unique. By default (`workflow_id_reuse_policy="return-existing"`), starting a workflow with an ID already in use returns the existing workflow (its handle, or its result for a direct call) without re-executing, whatever its status.

### Rejecting Reused Workflow IDs

To detect duplicates instead of silently attaching to the existing workflow, use `workflow_id_reuse_policy="reject"` (3.1+). It raises `DBOSWorkflowIDInUseError` without starting a new workflow or modifying the existing one.

**Incorrect (can't tell a new order from a duplicate submission):**

```python
with SetWorkflowID(f"order-{order_id}"):
    handle = DBOS.start_workflow(process_order, order)  # silently returns the old workflow
```

**Correct (reject duplicates explicitly):**

```python
from dbos import DBOS, SetWorkflowID
from dbos import error as dboserror

def submit_order(order_id: str, order: Order) -> str:
    try:
        with SetWorkflowID(f"order-{order_id}", workflow_id_reuse_policy="reject"):
            handle = DBOS.start_workflow(process_order, order)
        return handle.get_result()
    except dboserror.DBOSWorkflowIDInUseError as e:
        # e.workflow_id, e.workflow_status, e.workflow_name describe the existing workflow
        return "already submitted"
```

The same option is available as `"workflow_id_reuse_policy": "reject"` in `DBOSClient` `EnqueueOptions` (and `DBOS.enqueue_workflow_with_options`). It cannot be used with a debouncer.

Reference: [Workflow IDs and Idempotency](https://docs.dbos.dev/python/tutorials/workflow-tutorial#workflow-ids-and-idempotency)
