---
title: Run Database Migrations with dbos migrate
impact: CRITICAL
impactDescription: DBOS cannot function without its system tables for workflow state tracking
tags: migrate, system-tables, database, setup, production, app-role, permissions
---

## Run Database Migrations with dbos migrate

`dbos migrate` creates DBOS system tables and runs user-defined migration commands. DBOS auto-creates system tables on startup during development, but in production you should run migration separately with a privileged user and use `--app-role` for least-privilege access.

**Incorrect (application runs as admin in production):**

```bash
# App connects as admin — has access to everything
export DBOS_SYSTEM_DATABASE_URL=postgres://admin:pass@host/db
dbos start
```

**Correct (separate migration and runtime roles):**

```bash
# Step 1: Admin creates system tables and grants permissions
dbos migrate --sys-db-url postgres://admin:pass@host/db --app-role myapp_role

# Step 2: Application runs with limited role
export DBOS_SYSTEM_DATABASE_URL=postgres://myapp_role:pass@host/db
dbos start
```

The `--app-role` flag grants the specified role:
- SELECT, INSERT, UPDATE, DELETE on DBOS system tables
- USAGE on the DBOS schema
- No DDL permissions (cannot CREATE/DROP tables)

### Python

```bash
dbos migrate
dbos migrate --sys-db-url postgres://admin:pass@localhost:5432/mydb
dbos migrate --sys-db-url postgres://admin:pass@host/db --app-role myapp_role
```

| Flag | Short | Description |
|------|-------|-------------|
| `--sys-db-url <url>` | `-s` | System database connection string |
| `--db-url <url>` | `-D` | Application database connection string |
| `--app-role <role>` | `-r` | Grant minimum permissions to this role |
| `--config <file>` | | Config file path (default: `dbos-config.yaml`) |

Python's `dbos migrate` focuses on system tables. Application schema migrations are managed externally (e.g., Alembic).

### TypeScript

TypeScript uses two separate commands:

```bash
# Create DBOS system tables
npx dbos schema postgres://admin:pass@localhost:5432/mydb
npx dbos schema postgres://admin:pass@host/db --app-role myapp_role

# Run user-defined migration commands from dbos-config.yaml
npx dbos migrate
```

| Flag | Short | Description |
|------|-------|-------------|
| `systemDatabaseUrl` | (positional) | Required. System database URL |
| `--app-role <role>` | `-r` | Grant minimum permissions to this role |

User-defined migrations are configured in `dbos-config.yaml`:

```yaml
database:
  app_db_client: knex
migrate:
  - npx knex migrate:latest
```

### Go

Go's `dbos migrate` creates system tables and runs user-defined migration commands in a single step:

```bash
dbos migrate
dbos migrate --db-url postgres://admin:pass@localhost:5432/mydb
dbos migrate --db-url postgres://admin:pass@host/db --app-role myapp_role
```

| Flag | Short | Description |
|------|-------|-------------|
| `--db-url <url>` | `-D` | System database URL |
| `--app-role <role>` | `-r` | Grant minimum permissions to this role |
| `--schema <name>` | | Database schema (default: `dbos`) |
| `--config <file>` | | Config file path (default: `dbos-config.yaml`) |
| `--verbose` | | Enable DEBUG logging |

User-defined migrations in Go config:

```yaml
database:
  migrate:
    - go run ./cmd/migrate up
```

Database URL resolution priority: `--db-url` flag > `database_url` in config > `DBOS_SYSTEM_DATABASE_URL` env var.

Reference: [Python CLI](https://docs.dbos.dev/python/reference/cli) | [TypeScript CLI](https://docs.dbos.dev/typescript/reference/cli) | [Go CLI](https://docs.dbos.dev/golang/reference/cli)
