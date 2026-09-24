---
title: Tag Workflows with Searchable Attributes
impact: MEDIUM
impactDescription: Find workflows by business metadata (customer, tenant, region) without encoding it in workflow IDs
tags: workflow, attributes, SetWorkflowAttributes, update_workflow_attributes, list_workflows, schedule_name, metadata
---

## Tag Workflows with Searchable Attributes

Attach a dictionary of JSON-serializable key-value attributes to workflows with `SetWorkflowAttributes`, then search for them with the `attributes=` filter.

**Incorrect (encoding metadata in workflow IDs and scanning):**

```python
import uuid
from dbos import DBOS, SetWorkflowID

# process_order is a workflow defined elsewhere

with SetWorkflowID(f"acme-us-east-1-{uuid.uuid4()}"):
    process_order(order)

# Fragile prefix matching, can't query by region alone
acme = DBOS.list_workflows(workflow_id_prefix="acme-")
```

**Correct (attributes):**

```python
from dbos import DBOS, SetWorkflowAttributes

# Every workflow started or enqueued inside the block gets these attributes
with SetWorkflowAttributes({"customer": "acme", "region": "us-east-1"}):
    process_order(order)

# Matches workflows whose attributes contain ALL given key-value pairs
acme = DBOS.list_workflows(attributes={"customer": "acme"})
queued = DBOS.list_queued_workflows(attributes={"region": "us-east-1"})

for w in acme:
    print(w.workflow_id, w.attributes)
```

- Attributes are recorded at creation time and are **not** inherited by child workflows
- Filtering by `attributes` requires a Postgres system database (GIN-indexed JSONB); it raises `DBOSException` on SQLite
- Nested values are matched exactly

### Updating Attributes

`DBOS.update_workflow_attributes` (and `_async`) **replaces** the whole dictionary (not a merge); pass `None` to clear. It is safe to call from within a workflow, including on itself (the update is recorded as a step):

```python
@DBOS.workflow()
def process_order(order):
    reserve_inventory(order)  # reserve_inventory / ship are steps defined elsewhere
    DBOS.update_workflow_attributes(DBOS.workflow_id, {"customer": "acme", "phase": "shipping"})
    ship(order)
```

From outside the application:
- `DBOSClient` has `client.update_workflow_attributes(workflow_id, attrs)`, `list_workflows(attributes=...)`, and `list_queued_workflows(attributes=...)`
- Set attributes at enqueue time with `"attributes": {...}` in `EnqueueOptions`

### Schedule Name Tagging

Every workflow enqueued by a schedule is tagged with its schedule name (`WorkflowStatus.schedule_name`). Retrieve all runs of a schedule with:

```python
runs = DBOS.list_workflows(schedule_name="my-task-schedule")  # or a list of names
```

Reference: [Workflow Attributes](https://docs.dbos.dev/python/tutorials/workflow-management#workflow-attributes)
