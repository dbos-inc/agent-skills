---
name: dbos-typescript
description: DBOS TypeScript SDK for building reliable, fault-tolerant applications with durable workflows. Use this skill when writing TypeScript code with DBOS, creating workflows and steps, using queues, using DBOSClient from external applications, or building applications that need to be resilient to failures.
license: MIT
metadata:
  author: dbos
  version: "2.2.0"
  organization: DBOS
  date: October 2026
  abstract: Comprehensive guide for building fault-tolerant TypeScript applications with DBOS. Covers workflows, steps, queues, communication patterns, and best practices for durable execution.
---

# DBOS TypeScript Best Practices

Guide for building reliable, fault-tolerant TypeScript applications with DBOS durable workflows. Targets DBOS TypeScript 5.x (`@dbos-inc/dbos-sdk`). If a project is on 4.x or uses removed APIs (`new WorkflowQueue`, `@DBOS.scheduled`/`DBOS.registerScheduled`, `partitionQueue`, `priorityEnabled`, `@DBOS.requiredRole`, `dbos-config.yaml`-based config), see `references/advanced-upgrading.md`.

## When to Apply

Reference these guidelines when:
- Adding DBOS to existing TypeScript code
- Creating workflows and steps
- Using queues for concurrency control
- Implementing workflow communication (events, messages, streams)
- Configuring and launching DBOS applications
- Using DBOSClient from external applications
- Testing DBOS applications

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

### Installation

Always install the latest version of DBOS:

```bash
npm install @dbos-inc/dbos-sdk@latest
```

### DBOS Configuration and Launch

A DBOS application MUST configure and launch DBOS before running any workflows:

```typescript
import { DBOS } from "@dbos-inc/dbos-sdk";

async function main() {
  DBOS.setConfig({
    name: "my-app",
    applicationVersion: "0.1.0",
    systemDatabaseUrl: process.env.DBOS_SYSTEM_DATABASE_URL,
  });
  await DBOS.launch();
  await myWorkflow();
}

main().catch(console.log);
```

When creating a new application, set `applicationVersion` to `"0.1.0"`. If omitted, DBOS derives an opaque hash from workflow source code. When editing an existing application, leave its configured version alone — changing it is a deployment decision (see `references/advanced-versioning.md`).

### Workflow and Step Structure

Workflows are comprised of steps. Any function performing complex operations or accessing external services must be run as a step using `DBOS.runStep`:

```typescript
import { DBOS } from "@dbos-inc/dbos-sdk";

async function fetchData() {
  return await fetch("https://api.example.com").then(r => r.json());
}

async function myWorkflowFn() {
  const result = await DBOS.runStep(fetchData, { name: "fetchData" });
  return result;
}
const myWorkflow = DBOS.registerWorkflow(myWorkflowFn);
```

### Connecting to Conductor

DBOS applications run without Conductor. Add it when the user asks for it, or for what it provides: workflow recovery across processes, observability, alerting, visualization, managing workflows from a console, or the features of DBOS Transact Enterprise, such as suspending workflows, auto-upgrade and large object storage. Conductor is either DBOS-hosted (https://console.dbos.dev) or self-hosted on the user's own infrastructure. The choice is the user's: if they haven't made it, ask, describing both neutrally (see `references/lifecycle-config.md`), and don't recommend either. Users who don't want workflow data to leave their infrastructure often self-host.

Write the launch options so the same code works with either, leaving the choice to the environment:

```typescript
await DBOS.launch({
  conductorKey: process.env.DBOS_CONDUCTOR_KEY, // unset: no Conductor
  conductorURL: process.env.DBOS_CONDUCTOR_URL, // unset: DBOS-hosted Conductor
});
```

- DBOS-hosted: register the app and create an API key at https://console.dbos.dev, then set `DBOS_CONDUCTOR_KEY`
- Self-hosted: deploy Conductor with the `dbos-conductor-self-hosting` skill, register the app and create an API key in that Conductor's console, then set `DBOS_CONDUCTOR_KEY` and `DBOS_CONDUCTOR_URL` (its WebSocket URL, such as `ws://localhost:8090/`)

### Key Constraints

- Do NOT call, start, or enqueue workflows from within steps
- Do NOT use threads or uncontrolled concurrency to start workflows - use `DBOS.startWorkflow` or queues
- Workflows MUST be deterministic - non-deterministic operations go in steps
- Do NOT modify global variables from workflows or steps
- Register queues (`DBOS.registerQueue`) and create schedules (`DBOS.applySchedules`/`DBOS.createSchedule`) AFTER `DBOS.launch()` - they are stored in the system database
- Steps are only durable when called from a workflow; outside a workflow they run as plain function calls
- Connecting to DBOS Conductor (`conductorKey`, or deploying to DBOS Cloud), and auto-upgrade (`autoUpgrade`) require installing `@dbos-inc/dbos-enterprise` at the same minor version as `@dbos-inc/dbos-sdk`; auto-upgrade also needs an entitled key (`conductorKey`, or `transactEnterpriseKey` in development). See `references/lifecycle-config.md`
- Do NOT set `enablePatching` (deprecated in 5.4): `DBOS.patch()` and `DBOS.deprecatePatch()` work without it. To let patched code recover older workflows, pin `applicationVersion` or use auto-upgrade (see `references/advanced-patching.md`)

## How to Use

Read individual rule files for detailed explanations and examples:

```
references/lifecycle-config.md
references/workflow-determinism.md
references/queue-concurrency.md
references/advanced-upgrading.md
references/advanced-shared-database.md
```

If multiple applications share one system database, always set `applicationName` on `DBOSClient` and see `references/advanced-shared-database.md`.

## References

- https://docs.dbos.dev/
- https://github.com/dbos-inc/dbos-transact-ts
