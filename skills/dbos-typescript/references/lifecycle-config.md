---
title: Configure and Launch DBOS Properly
impact: CRITICAL
impactDescription: Application won't function without proper setup
tags: configuration, launch, setup, initialization, migrate, conductor, enterprise, auto-upgrade
---

## Configure and Launch DBOS Properly

Every DBOS application must configure and launch DBOS before running any workflows. All workflows and steps must be registered before calling `DBOS.launch()`.

**Incorrect (missing configuration or launch):**

```typescript
import { DBOS } from "@dbos-inc/dbos-sdk";

// No configuration or launch!
async function myWorkflowFn() {
  // This will fail - DBOS is not launched
}
const myWorkflow = DBOS.registerWorkflow(myWorkflowFn);
await myWorkflow();
```

**Correct (configure and launch in main):**

```typescript
import { DBOS } from "@dbos-inc/dbos-sdk";

async function myWorkflowFn() {
  // workflow logic
}
const myWorkflow = DBOS.registerWorkflow(myWorkflowFn);

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

For scheduled-only applications, create schedules after launch:

```typescript
async function main() {
  DBOS.setConfig({ name: "my-app", applicationVersion: "0.1.0", systemDatabaseUrl: process.env.DBOS_SYSTEM_DATABASE_URL });
  await DBOS.launch();
  await DBOS.applySchedules([
    { scheduleName: "my-task", workflowFn: scheduledTask, schedule: "* * * * *" },
  ]);
}
```

## DBOSConfig Reference

All fields except `name` are optional. `DBOS.launch()` throws `DBOSInitializationError` if no configuration or no `name` was provided. DBOS does not read `dbos-config.yaml` at launch (the DBOS CLI and DBOS Cloud still use it); always configure with `DBOS.setConfig`.

`name` identifies which application owns each workflow, queue, schedule, and application version in the system database. Applications sharing a system database must have distinct names, and a process only runs its own application's workflows.

| Field | Description | Default |
|-------|-------------|---------|
| **name** | Application name and ownership key | (required) |
| **applicationVersion** | Version tag for versioning strategy. Set to `"0.1.0"` in new applications | Auto-computed hash |
| **executorID** | Unique process ID for distributed environments | Auto-set by Conductor/Cloud |
| **enablePatching** | Deprecated (5.4): `DBOS.patch()` no longer needs it; it only pins `applicationVersion` to `PATCHING_ENABLED` and logs a warning. Set `applicationVersion` instead | — |
| **transactEnterpriseKey** | Key entitling the app to DBOS Transact Enterprise features without Conductor. Development only; in production, pass `conductorKey` to `DBOS.launch` (it stands in for this key) | `undefined` |
| **autoUpgrade** | Auto-upgrade every workflow except those registered with `autoUpgrade: false` (see `advanced-patching.md`). Requires DBOS Transact Enterprise | `false` |
| **autoUpgradeIntervalMs** | Milliseconds between auto-upgrade passes | `30000` |
| **autoUpgradeGraceMs** | Milliseconds the latest version must have been the latest before `PENDING` workflows on older versions are moved (`ENQUEUED`/`DELAYED` move without waiting) | `0` |
| **systemDatabaseUrl** | Postgres connection string for system DB | `postgresql://postgres:dbos@localhost:5432/[name]_dbos_sys` |
| **systemDatabasePoolSize** | System DB connection pool size | `10` |
| **systemDatabasePollingConcurrency** | Max concurrent database-backed polling reads from wait operations (`getResult`, `waitFirst`, `recv`, `getEvent`, ...), so high-fan-out polling can't starve enqueue/dequeue, status writes, recovery, and cancellation. Non-positive disables the limit | Half the pool size (min 1) |
| **systemDatabaseSchemaName** | Postgres schema for DBOS system tables | `"dbos"` |
| **systemDatabasePool** | Custom `node-postgres` pool (skips pool creation; you own it, `shutdown` doesn't close it) | `undefined` |
| **runMigrations** | Create/migrate the system database on launch. Set `false` if the role can't run DDL and you migrate with `npx dbos schema` or `DBOS.migrate`; launch then only verifies the schema | `true` |
| **observabilityQueryTimeoutMs** | Statement timeout for `listWorkflows`, `listQueuedWorkflows`, `listWorkflowSteps`, etc.; exceeding it throws `DBOSQueryTimeoutError`. `<= 0` disables | `30000` |
| **systemDatabaseIdleTransactionTimeoutMs** | Postgres `idle_in_transaction_session_timeout` set on the system database connections DBOS creates | `60000` |
| **useListenNotify** | Use Postgres `LISTEN/NOTIFY` to wake `recv`/`getEvent`/`readStream` waiters. Set `false` if unsupported (e.g., CockroachDB) to poll instead | `true` |
| **notificationCoalesceMs** | With `useListenNotify`, interval over which event/stream notifications are batched | `10` |
| **tracingEnabled** | Generate DBOS traces for an external OpenTelemetry `TracerProvider` | — |
| **otelAttributeFormat** | Span attribute naming: `'legacy'` or `'semconv'` (`dbos.*` names) | `'legacy'` |
| **enableOTLP** | Enable the built-in DBOS OpenTelemetry `TracerProvider` and export | `false` (`true` in DBOS Cloud) |
| **otlpTracesEndpoints** | OTLP trace receiver URLs (built-in provider only) | `undefined` |
| **otlpLogsEndpoints** | OTLP log receiver URLs (built-in provider only) | `undefined` |
| **logLevel** | DBOS logger severity | `"info"` |
| **logger** | Custom logger implementing the `DLogger` interface, to which DBOS directs all its internal logging, replacing the built-in console and OTLP log sinks. `error()` receives the `Error`'s message, with its stack (including its `cause` chain) in `metadata.stack` and the original `Error` in `metadata.error` | `undefined` |
| **addContextMetadata** | Append workflow ID/operation name to console logs from workflows and steps (only when `enableOTLP` is on) | `false` |
| **listenQueues** | Names of the only queues this process dequeues from (`string[]`). Names not matching a queue at launch are picked up when registered | All queues owned by this application |
| **maxConcurrentQueueDispatches** | Max number of queues this process dequeues from concurrently | `3` |
| **schedulerPollingIntervalMs** | Scheduler polling interval for new schedules (ms) | `30000` |
| **serializer** | Custom serializer for system database | Default (SuperJSON) |

## Connecting to Conductor

Conductor runs in one of two places. Which one is the user's choice: if they haven't said, ask, presenting both as below, and don't recommend either.

| | DBOS-hosted | Self-hosted |
|---|---|---|
| Who runs it | DBOS, at https://console.dbos.dev | The user, on their own infrastructure (see the `dbos-conductor-self-hosting` skill) |
| Setup | Register the app and create an API key in the DBOS Console | Deploy Conductor, then register the app and create an API key in its console |
| Configuration | `conductorKey` | `conductorKey` and `conductorURL` |
| Workflow data | Workflow metadata goes to DBOS. Workflow data such as inputs and outputs is also sent, for display in the console, unless metadata-only mode is on (`conductorMetadataOnlyMode`; DBOS Teams plan or above). Conductor never stores application data or connects to the app's database | Stays on the user's infrastructure. Conductor contacts DBOS only to validate its license at startup, and not at all with an offline license key (for air-gapped environments; the user requests one from contact@dbos.dev) |
| License | Covered by the user's DBOS plan | A self-serve trial key to evaluate; for production, a license key the user requests from contact@dbos.dev |

Pass Conductor settings to `DBOS.launch` (not `DBOS.setConfig`). Either way, connecting requires the `@dbos-inc/dbos-enterprise` package at the same minor version as `@dbos-inc/dbos-sdk` (upgrade the two together); without it, `DBOS.launch` throws `DBOSInitializationError` when `conductorKey` is set or the app runs in DBOS Cloud:

```shell
npm install @dbos-inc/dbos-enterprise
```

Read both settings from the environment so the same code serves either choice. With `DBOS_CONDUCTOR_URL` unset (or empty), the app connects to DBOS-hosted Conductor; with `DBOS_CONDUCTOR_KEY` unset (or empty), it doesn't connect at all:

```typescript
await DBOS.launch({
  conductorKey: process.env.DBOS_CONDUCTOR_KEY,
  conductorURL: process.env.DBOS_CONDUCTOR_URL,
  // conductorExecutorMetadata: { region: "us-east-1" },
  // conductorMetadataOnlyMode: true, // never send workflow data, only metadata
});
```

For a self-hosted Conductor, `DBOS_CONDUCTOR_URL` is its WebSocket URL: `ws://localhost:8090/` for a local trial, or the `wss://` URL of its TLS proxy in production. The API key comes from that Conductor's console, not https://console.dbos.dev.

The same package provides the DBOS Transact Enterprise feature auto-upgrade (`autoUpgrade`, see `advanced-patching.md`). It also requires a key whose organization is entitled to it, which `DBOS.launch` checks with Conductor: the `conductorKey` in production or, in development without Conductor, `transactEnterpriseKey` in `DBOS.setConfig` (a Conductor API key from the DBOS Console). A missing package, key, or entitlement makes `DBOS.launch` throw `DBOSInitializationError`.

## Least-Privilege Deployment

By default, `DBOS.launch()` creates the system database and migrates its tables, which requires DDL privileges. If the app's database role can't (or shouldn't) run DDL, migrate out of band with a privileged user and set `runMigrations: false`:

```shell
# As a privileged user: create/migrate the system tables and grant the app role access
npx dbos schema ${DBOS_SYSTEM_DATABASE_URL} -r my_app_role

# Or emit SQL for a DBA to apply (never connects; must run outside a transaction block)
npx dbos schema --print-migrations all ${DBOS_SYSTEM_DATABASE_URL} > migrations.sql
npx dbos schema --print-user-role -r my_app_role ${DBOS_SYSTEM_DATABASE_URL} > grants.sql
```

Or run the same migration from code (for example, in a deployment migration script). It creates the database if it does not exist:

```typescript
// Run with a privileged role
await DBOS.migrate(process.env.ADMIN_SYSTEM_DATABASE_URL!, {
  applicationRole: "my_app_role", // grant this role access to the DBOS schema
  // schemaName: "dbos",
});
```

```typescript
DBOS.setConfig({
  name: "my-app",
  systemDatabaseUrl: process.env.DBOS_SYSTEM_DATABASE_URL, // connects as my_app_role
  runMigrations: false,
});
await DBOS.launch();
```

- `-s, --schema <name>` targets a non-default schema (match `systemDatabaseSchemaName`)
- `--print-migrations <all|NUMBER>` prints all migrations (fresh database) or those from a migration number (upgrade)
- With `runMigrations: false`, launch only verifies the schema: a missing or outdated system database fails with `DBOSInitializationError`; a newer schema is accepted
- `npx dbos schema` and `DBOS.migrate` do not migrate datasource tables; see `step-transactions.md`

Reference: [DBOS Configuration](https://docs.dbos.dev/typescript/reference/configuration)
