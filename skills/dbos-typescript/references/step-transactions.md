---
title: Use Transactions for Database Operations
impact: HIGH
impactDescription: Transactions provide exactly-once database execution within workflows
tags: step, transaction, database, datasource, migrate, runMigrations, initializeDBOSSchema
---

## Use Transactions for Database Operations

Use datasource transactions for database operations within workflows. Transactions commit exactly once and are checkpointed for recovery.

**Incorrect (raw database query in workflow):**

```typescript
import { Pool } from "pg";
const pool = new Pool();

async function myWorkflowFn() {
  // Direct database access in workflow - not checkpointed!
  const result = await pool.query("INSERT INTO orders ...");
}
```

**Correct (using a datasource transaction):**

Install a datasource package (e.g., Knex):
```
npm i @dbos-inc/knex-datasource
```

Configure the datasource:
```typescript
import { KnexDataSource } from "@dbos-inc/knex-datasource";

const config = { client: "pg", connection: process.env.DBOS_DATABASE_URL };
const dataSource = new KnexDataSource("app-db", config);
```

Run transactions inline with `runTransaction`:
```typescript
async function insertOrderFn(userId: string, amount: number) {
  const rows = await dataSource
    .client("orders")
    .insert({ user_id: userId, amount })
    .returning("id");
  return rows[0].id;
}

async function myWorkflowFn(userId: string, amount: number) {
  const orderId = await dataSource.runTransaction(
    () => insertOrderFn(userId, amount),
    { name: "insertOrder" }
  );
  return orderId;
}
const myWorkflow = DBOS.registerWorkflow(myWorkflowFn);
```

You can also pre-register a transaction function with `dataSource.registerTransaction`:
```typescript
const insertOrder = dataSource.registerTransaction(insertOrderFn);
```

Available datasource packages: `@dbos-inc/knex-datasource`, `@dbos-inc/kysely-datasource`, `@dbos-inc/drizzle-datasource`, `@dbos-inc/typeorm-datasource`, `@dbos-inc/prisma-datasource`, `@dbos-inc/node-pg-datasource`, `@dbos-inc/postgres-datasource`.

Transaction config (e.g., Knex `{ isolationLevel, readOnly }`) is passed as the second argument to `runTransaction`/`registerTransaction`. `isolationLevel` defaults to the database's default (`read committed` in Postgres).

### Datasource Schema and Migrations

Datasources record each transaction in a `transaction_completion` table (in the `dbos` schema by default) so it runs exactly once; once a workflow completes, DBOS deletes its rows. By default, each datasource creates or migrates this table when DBOS launches, which requires DDL privileges in your database.

If the app's database role can't run DDL, install the schema with the datasource's static `initializeDBOSSchema` method using a privileged role (e.g., in your schema migrations), then construct the datasource with `runMigrations: false` in its final options argument so launch only verifies the schema (throwing `DBOSInitializationError` if it is missing or outdated):

```typescript
// Migration script, run with a privileged role.
// applicationRole is granted USAGE on the schema, SELECT/INSERT/DELETE on transaction_completion,
// and SELECT on the dbos_transaction_completion_migrations version table.
await KnexDataSource.initializeDBOSSchema(adminConfig, "dbos", { applicationRole: "my_app_role" });

// Application, running as my_app_role:
const dataSource = new KnexDataSource("app-db", config, "dbos", { runMigrations: false });
```

`npx dbos schema` and `DBOS.migrate` migrate only the system database, not datasource tables.

Reference: [Transactions & Datasources](https://docs.dbos.dev/typescript/tutorials/transaction-tutorial)
