---
title: Define Migration and Rollback Commands in Config
impact: HIGH
impactDescription: Migration commands must be correctly configured for schema changes to apply
tags: configuration, migrate, rollback, dbos-config, schema
---

## Define Migration and Rollback Commands in Config

The `migrate` and `rollback` sections in `dbos-config.yaml` define shell commands that `dbos migrate` and `dbos rollback` execute.

**Incorrect (no migrate commands in config):**

```yaml
# dbos-config.yaml — missing migrate section, npx dbos migrate does nothing
name: my-app
language: node
database:
  hostname: localhost
  port: 5432
  username: postgres
  password: ${PGPASSWORD}
  app_db_client: knex
```

**Correct (migrate and rollback defined):**

```yaml
# dbos-config.yaml
name: my-app
language: node
database:
  hostname: localhost
  port: 5432
  username: postgres
  password: ${PGPASSWORD}
  app_db_client: knex
migrate:
  - npx knex migrate:latest
rollback:
  - npx knex migrate:rollback
```

### TypeScript

```yaml
name: my-app
language: node
database:
  hostname: localhost
  port: 5432
  username: postgres
  password: ${PGPASSWORD}
  app_db_client: knex
migrate:
  - npx knex migrate:latest
rollback:
  - npx knex migrate:rollback
```

Multiple commands run sequentially:

```yaml
migrate:
  - npx prisma migrate deploy
  - npx prisma generate
```

### Go

```yaml
name: my-app
database_url: ${DBOS_SYSTEM_DATABASE_URL}
database:
  migrate:
    - go run ./cmd/migrate up
```

Note: In Go, the `migrate` key is nested under `database`.

### Python (Cloud Deployment)

```yaml
name: my-app
language: python
migrate:
  - alembic upgrade head
```

Python's `dbos migrate` CLI command does not run these commands locally — it only creates system tables. The `migrate` section is used by DBOS Cloud during deployment.

### Runtime Configuration

The `runtimeConfig` section defines start and setup commands:

```yaml
runtimeConfig:
  start:
    - npm start          # TypeScript
    - fastapi run        # Python
    - go run .           # Go
  setup:                 # Pre-build (DBOS Cloud only)
    - npm install
```

Reference: [TS Config](https://docs.dbos.dev/typescript/reference/configuration) | [Deploying to Cloud](https://docs.dbos.dev/production/dbos-cloud/deploying-to-cloud)
