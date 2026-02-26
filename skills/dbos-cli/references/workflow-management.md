---
title: Manage Workflows from the Command Line
impact: MEDIUM
impactDescription: Enables operational control over running and failed workflows
tags: workflow, list, cancel, resume, fork, inspect, operations
---

## Manage Workflows from the Command Line

The DBOS CLI provides commands to list, inspect, cancel, resume, and fork workflows. These commands are available in Python, TypeScript, and Go.

**Incorrect (restarting the app to retry a failed workflow):**

```bash
# Workflow failed due to transient error — restarting the whole app is overkill
dbos reset
dbos migrate
dbos start  # Loses all workflow state
```

**Correct (resume just the failed workflow):**

```bash
# Inspect the failure
dbos workflow get <workflow-id>
dbos workflow steps <workflow-id>

# Resume from last completed step
dbos workflow resume <workflow-id>
```

### List Workflows

```bash
# Python
dbos workflow list

# TypeScript
npx dbos workflow list

# Go
dbos workflow list
```

Output is JSON. Filter with `jq`:

```bash
dbos workflow list | jq '.[] | select(.status == "ERROR")'
```

### Inspect a Workflow

```bash
# Get workflow details
dbos workflow get <workflow-id>

# List steps executed by a workflow
dbos workflow steps <workflow-id>
```

### Cancel a Workflow

Cancels a running or pending workflow:

```bash
dbos workflow cancel <workflow-id>
```

### Resume a Workflow

Resumes a workflow from its last completed step:

```bash
dbos workflow resume <workflow-id>
```

Use this to retry a workflow that failed due to a transient error (e.g., external service was down).

### Fork a Workflow

Creates a new workflow execution starting from a specific step:

```bash
dbos workflow fork <workflow-id>
```

### Queue Management

```bash
# List enqueued workflows/tasks
dbos workflow queue list         # Python
npx dbos workflow queue list     # TypeScript
```

### Command Summary

| Command | Python | TypeScript | Go |
|---------|--------|------------|-----|
| List | `dbos workflow list` | `npx dbos workflow list` | `dbos workflow list` |
| Get | `dbos workflow get <id>` | `npx dbos workflow get <id>` | `dbos workflow get <id>` |
| Steps | `dbos workflow steps <id>` | `npx dbos workflow steps <id>` | `dbos workflow steps <id>` |
| Cancel | `dbos workflow cancel <id>` | `npx dbos workflow cancel <id>` | `dbos workflow cancel <id>` |
| Resume | `dbos workflow resume <id>` | `npx dbos workflow resume <id>` | `dbos workflow resume <id>` |
| Fork | `dbos workflow fork <id>` | `npx dbos workflow fork <id>` | `dbos workflow fork <id>` |
| Queue list | `dbos workflow queue list` | `npx dbos workflow queue list` | — |

Reference: [Python CLI](https://docs.dbos.dev/python/reference/cli)
