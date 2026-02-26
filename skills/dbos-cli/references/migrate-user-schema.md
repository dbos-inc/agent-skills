---
title: Run User-Defined Schema Migrations
impact: CRITICAL
impactDescription: Application schema must be applied before the app can read/write data
tags: migrate, schema, knex, alembic, user-migrations, rollback
---

## Run User-Defined Schema Migrations

User-defined migrations apply your application's database schema (tables, indexes, etc.). These are configured in `dbos-config.yaml` and executed by the CLI.

**Incorrect (running migrations manually, not tracked in config):**

```bash
# Won't run automatically on deploy — migration is ad-hoc
npx knex migrate:latest
```

**Correct (define migrations in dbos-config.yaml):**

```yaml
# dbos-config.yaml
migrate:
  - npx knex migrate:latest
```

```bash
npx dbos migrate  # Runs the configured migration commands
```

### TypeScript

TypeScript has a dedicated `npx dbos migrate` command that runs the `migrate` commands from config:

```yaml
# dbos-config.yaml
database:
  app_db_client: knex
migrate:
  - npx knex migrate:latest
rollback:
  - npx knex migrate:rollback
```

```bash
# Run all migration commands
npx dbos migrate

# Roll back the last batch
npx dbos rollback
```

Supported ORMs via `app_db_client`: `knex`, `prisma`, `drizzle`, `typeorm`.

**Knex configuration tip** — load database settings from DBOS config to avoid duplication:

```javascript
// knexfile.js
const { parseConfigFile } = require("@dbos-inc/dbos-sdk");
const [dbosConfig] = parseConfigFile();

module.exports = {
  client: "pg",
  connection: {
    host: dbosConfig.poolConfig.host,
    port: dbosConfig.poolConfig.port,
    user: dbosConfig.poolConfig.user,
    password: dbosConfig.poolConfig.password,
    database: dbosConfig.poolConfig.database,
  },
};
```

### Go

Go's `dbos migrate` runs both system table creation and user-defined migration commands from config in a single step:

```yaml
# dbos-config.yaml
database:
  migrate:
    - echo "Running migrations..."
    - go run ./migrations
```

```bash
dbos migrate
```

### Python

Python's `dbos migrate` focuses on system tables. Application schema migrations are managed externally with tools like Alembic:

```bash
# System tables
dbos migrate

# Application schema (separate tool)
alembic upgrade head
```

For DBOS Cloud deployment, you can specify migration commands in config that run during deploy:

```yaml
# dbos-config.yaml
migrate:
  - alembic upgrade head
```

Reference: [Using Knex](https://docs.dbos.dev/typescript/tutorials/orms/using-knex) | [Python CLI](https://docs.dbos.dev/python/reference/cli)
