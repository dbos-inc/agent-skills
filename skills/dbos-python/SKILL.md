---
name: dbos-python
description: DBOS Python SDK (3.x) for building reliable, fault-tolerant applications with durable workflows. Use this skill when writing Python code with DBOS, creating workflows and steps, using queues, datasource transactions, or schedules, using DBOSClient from external applications, upgrading DBOS Python 2.x code to 3.x, or building applications that need to be resilient to failures.
license: MIT
metadata:
  author: dbos
  version: "2.0.0"
  organization: DBOS
  date: September 2026
  abstract: Comprehensive guide for building fault-tolerant Python applications with DBOS. Covers workflows, steps, queues, communication patterns, and best practices for durable execution.
---

# DBOS Python Best Practices

Guide for building reliable, fault-tolerant Python applications with DBOS durable workflows. Targets DBOS Python 3.x.

## When to Apply

Reference these guidelines when:
- Adding DBOS to existing Python code
- Creating workflows and steps
- Using queues for concurrency control
- Implementing workflow communication (events, messages, streams)
- Configuring and launching DBOS applications
- Using DBOSClient from external applications
- Testing DBOS applications
- Upgrading DBOS Python 2.x code to 3.x

## Rule Categories by Priority

| Priority | Category | Impact | Prefix |
|----------|----------|--------|--------|
| 1 | Lifecycle | CRITICAL | `lifecycle-` |
| 2 | Workflow | CRITICAL | `workflow-` |
| 3 | Step | HIGH | `step-` |
| 4 | Queue | HIGH | `queue-` |
| 5 | Communication | MEDIUM | `comm-` |
| 6 | Pattern | MEDIUM | `pattern-` |
| 7 | Testing | LOW-MEDIUM | `test-` |
| 8 | Client | MEDIUM | `client-` |
| 9 | Advanced | LOW | `advanced-` |

## Critical Rules

### DBOS Configuration and Launch

A DBOS application MUST configure and launch DBOS inside its main function:

```python
import os
from dbos import DBOS, DBOSConfig

@DBOS.workflow()
def my_workflow():
    pass

if __name__ == "__main__":
    config: DBOSConfig = {
        "name": "my-app",
        "application_version": "0.1.0",
        "system_database_url": os.environ.get("DBOS_SYSTEM_DATABASE_URL"),
    }
    DBOS(config=config)
    DBOS.launch()
```

When creating a new application, set `application_version` to `"0.1.0"`. If omitted, DBOS derives an opaque hash from workflow source code. When editing an existing application, leave its configured version alone — changing it is a deployment decision (see `references/advanced-versioning.md`).

### Workflow and Step Structure

Workflows are comprised of steps. Any function performing complex operations or accessing external services must be a step:

```python
@DBOS.step()
def call_external_api():
    return requests.get("https://api.example.com").json()

@DBOS.workflow()
def my_workflow():
    result = call_external_api()
    return result
```

### Key Constraints

- Do NOT call `DBOS.start_workflow` or `DBOS.recv` from a step
- Do NOT use threads to start workflows - use `DBOS.start_workflow` or queues
- Workflows MUST be deterministic - non-deterministic operations go in steps
- Do NOT create/update global variables from workflows or steps
- In `async def` code, use the `_async` variants of DBOS methods (`await DBOS.start_workflow_async(...)`, `send_async`, `recv_async`, `sleep_async`, `register_queue_async`, ...): many synchronous DBOS methods (such as `DBOS.sleep`, `DBOS.recv`, `DBOS.send`, `DBOS.set_event`, `DBOS.get_event`, and `DBOS.register_queue`) raise `RuntimeError` when called while an event loop is running
- Register queues and create schedules AFTER `DBOS.launch()`; create datasources BEFORE it

### Removed in DBOS 3.0 (never generate these)

`@DBOS.transaction` / `DBOS.sql_session` / `application_database_url` (use datasources), `Queue(...)` (use `DBOS.register_queue`), `partition_queue` / `priority_enabled`, `@DBOS.scheduled` (use `DBOS.apply_schedules`), and `DBOS(fastapi=...)` / `DBOS(flask=...)`. To migrate existing 2.x code, see `references/advanced-upgrading-v3.md`.

## How to Use

Read individual rule files for detailed explanations and examples:

```
references/lifecycle-config.md
references/workflow-determinism.md
references/queue-concurrency.md
references/advanced-upgrading-v3.md
references/advanced-shared-database.md
```

If multiple applications share one system database, see `references/advanced-shared-database.md`.

## References

- https://docs.dbos.dev/
- https://github.com/dbos-inc/dbos-transact-py
