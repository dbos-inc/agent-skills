---
title: Configure Database Connections in dbos-config.yaml
impact: HIGH
impactDescription: Incorrect database configuration prevents application startup and migrations
tags: configuration, database, connection, dbos-config, environment-variables
---

## Configure Database Connections in dbos-config.yaml

Database connection settings in `dbos-config.yaml` vary by language SDK.

### Python

```yaml
name: my-app
language: python
system_database_url: ${DBOS_SYSTEM_DATABASE_URL}
```

Python uses SQLite by default for the system database during development (no Postgres required). For production, set `system_database_url` to a Postgres connection string.

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
  system_database_schema_name: dbos
```

TypeScript supports `${ENV_VAR}` syntax for environment variable substitution. The `app_db_client` field determines which ORM is used: `knex`, `prisma`, `drizzle`, or `typeorm`.

### Go

```yaml
name: my-app
database_url: ${DBOS_SYSTEM_DATABASE_URL}
```

### Environment Variables

| Variable | Description | Languages |
|----------|-------------|-----------|
| `DBOS_SYSTEM_DATABASE_URL` | Postgres URL for DBOS system tables | Python, Go, TS |
| `DBOS_DATABASE_URL` | Application database URL (DBOS Cloud) | TS |
| `PGPASSWORD` | Postgres password | TS |

### Connection String Format

```
postgres://username:password@hostname:port/database_name
```

**Incorrect (missing components):**
```yaml
database_url: localhost/mydb  # Missing protocol, user, port
```

**Correct:**
```yaml
database_url: postgres://postgres:password@localhost:5432/mydb
```

Reference: [Python Config](https://docs.dbos.dev/python/reference/configuration) | [TypeScript Config](https://docs.dbos.dev/typescript/reference/configuration) | [Go Programming Guide](https://docs.dbos.dev/golang/programming-guide)
