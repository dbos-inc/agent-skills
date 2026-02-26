---
title: Create DBOS System Tables with Migrate
impact: CRITICAL
impactDescription: DBOS cannot function without its system tables for workflow state tracking
tags: migrate, system-tables, database, setup, production
---

## Create DBOS System Tables with Migrate

DBOS stores workflow state, step results, and metadata in system tables. The `dbos migrate` command creates these tables. By default, DBOS auto-creates system tables on application startup, but in production you should run migration separately with a privileged user.

**Incorrect (skipping migration, relying on auto-creation with limited user):**

```bash
# App runs as limited user — cannot create system tables
export DBOS_SYSTEM_DATABASE_URL=postgres://app_user:pass@host/db
dbos start  # Fails: permission denied for CREATE TABLE
```

**Correct (run migrate first with a privileged user):**

```bash
# Admin creates system tables
dbos migrate --sys-db-url postgres://admin:pass@host/db
# App starts successfully — tables already exist
export DBOS_SYSTEM_DATABASE_URL=postgres://app_user:pass@host/db
dbos start
```

**Python:**

```bash
# Uses DBOS_SYSTEM_DATABASE_URL env var or dbos-config.yaml
dbos migrate

# Explicit system database URL
dbos migrate --sys-db-url postgres://admin:pass@localhost:5432/mydb

# With both system and application databases
dbos migrate --sys-db-url postgres://admin:pass@localhost/sysdb \
             --db-url postgres://admin:pass@localhost/appdb
```

| Flag | Short | Description |
|------|-------|-------------|
| `--sys-db-url <url>` | `-s` | System database connection string |
| `--db-url <url>` | `-D` | Application database connection string |
| `--app-role <role>` | `-r` | Grant minimum permissions to this role |
| `--config <file>` | | Config file path (default: `dbos-config.yaml`) |

**TypeScript:**

TypeScript uses `npx dbos schema` (not `npx dbos migrate`) to create system tables:

```bash
# Create system tables (URL is required)
npx dbos schema postgres://admin:pass@localhost:5432/mydb

# With app role for least-privilege access
npx dbos schema postgres://admin:pass@localhost/mydb --app-role myapp_role
```

| Flag | Short | Description |
|------|-------|-------------|
| `systemDatabaseUrl` | (positional) | Required. System database URL |
| `--app-role <role>` | `-r` | Grant minimum permissions to this role |

**Go:**

```bash
# Uses --db-url flag, config file, or DBOS_SYSTEM_DATABASE_URL env var
dbos migrate

# Explicit database URL
dbos migrate --db-url postgres://admin:pass@localhost:5432/mydb

# Custom schema name
dbos migrate --schema custom_schema

# With app role
dbos migrate --app-role myapp_role
```

| Flag | Short | Description |
|------|-------|-------------|
| `--db-url <url>` | `-D` | System database URL |
| `--app-role <role>` | `-r` | Grant minimum permissions to this role |
| `--schema <name>` | | Database schema (default: `dbos`) |
| `--config <file>` | | Config file path (default: `dbos-config.yaml`) |
| `--verbose` | | Enable DEBUG logging |

**Database URL resolution priority (Go):**
1. `--db-url` flag
2. `database_url` in `dbos-config.yaml`
3. `DBOS_SYSTEM_DATABASE_URL` environment variable

Reference: [Python CLI](https://docs.dbos.dev/python/reference/cli) | [Go CLI](https://docs.dbos.dev/golang/programming-guide)
