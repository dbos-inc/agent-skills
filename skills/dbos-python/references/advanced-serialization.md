---
title: Use Portable Serialization for Cross-Language Interoperability
impact: LOW
impactDescription: Enables Python workflows to exchange data with TypeScript, Java, and Go applications
tags: serialization, portable, cross-language, interoperability, json, WorkflowSerializationFormat
---

## Use Portable Serialization for Cross-Language Interoperability

By default, Python DBOS uses pickle serialization, which only Python can read. Use `WorkflowSerializationFormat.PORTABLE` to serialize data as JSON that any DBOS SDK (Python, TypeScript, Java, Go) can read and write.

**Incorrect (default pickle — blocks cross-language access):**

```python
# A TypeScript or Java client cannot read this workflow's
# inputs, outputs, events, or streams
@DBOS.workflow()
def process_order(order_id: str):
    DBOS.set_event("status", {"progress": 50})
    return {"result": "done"}
```

**Correct (portable JSON — readable by any language):**

```python
from dbos import DBOS, WorkflowSerializationFormat

@DBOS.workflow(serialization_type=WorkflowSerializationFormat.PORTABLE)
def process_order(order_id: str):
    # Inputs, outputs, events, and streams default to portable JSON
    DBOS.set_event("status", {"progress": 50})
    return {"result": "done"}
```

### Supported Portable Types

Portable JSON supports JSON primitives, arrays, and objects. Some Python types are automatically converted:

| Python Type | Portable Representation |
|-------------|------------------------|
| `datetime` | RFC 3339 UTC string |
| `date` | ISO 8601 string |
| `Decimal` | Numeric string |
| `set`, `tuple` | JSON array |

### Where to Set Serialization

**On the workflow decorator** — affects inputs, outputs, events, and streams for that workflow:

```python
@DBOS.workflow(serialization_type=WorkflowSerializationFormat.PORTABLE)
def my_workflow(data: dict):
    DBOS.set_event("progress", 50)  # Portable by default
    return {"done": True}           # Portable by default
```

**On individual operations** — override per-operation when mixing strategies:

```python
# Explicitly set portable on send (send is never affected by workflow default)
DBOS.send(
    destination_id="workflow-123",
    message={"status": "complete"},
    topic="updates",
    serialization_type=WorkflowSerializationFormat.PORTABLE,
)

# Override on set_event or write_stream
DBOS.set_event("key", value, serialization_type=WorkflowSerializationFormat.PORTABLE)
DBOS.write_stream("key", value, serialization_type=WorkflowSerializationFormat.PORTABLE)
```

**On enqueue from DBOSClient** — for cross-language workflow submission:

```python
from dbos import DBOSClient, WorkflowSerializationFormat

client = DBOSClient(system_database_url=db_url)
handle = client.enqueue(
    {
        "workflow_name": "process_order",
        "queue_name": "orders",
        "serialization_type": WorkflowSerializationFormat.PORTABLE,
    },
    "order-123",
)
```

### Serialization Strategy Options

```python
from dbos import WorkflowSerializationFormat

WorkflowSerializationFormat.DEFAULT   # Uses config serializer (pickle by default)
WorkflowSerializationFormat.PORTABLE  # Portable JSON for cross-language use
WorkflowSerializationFormat.NATIVE    # Explicitly uses Python pickle (py_pickle)
```

### Portable Errors

When a portable workflow fails, the error is serialized in a standard JSON structure all languages can read:

```python
from dbos import PortableWorkflowError

raise PortableWorkflowError(
    message="Order not found",
    name="NotFoundError",
    code=404,
    data={"orderId": "order-123"},
)
```

Non-portable exceptions raised in a portable workflow are automatically converted to this format on a best-effort basis.

### Key Rules

- `send` is **never** affected by the workflow's serialization strategy — always set `serialization_type` explicitly on `send` for cross-language messages
- Step outputs always use the native serializer regardless of workflow strategy (steps are internal)
- `DBOSClient.serializer` must match the app's serializer for **default**-format data, but portable data is always readable

Reference: [Cross-Language Interaction](https://docs.dbos.dev/explanations/portable-workflows)
