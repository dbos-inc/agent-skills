---
name: dbos-cli
description: >
  DBOS CLI for database migrations, workflow management, and local development
  across Python, TypeScript, and Go. Use this skill when running dbos migrate,
  managing DBOS system tables, setting up database permissions, initializing
  DBOS projects, managing workflows from the command line, or working with
  dbos-config.yaml.
license: MIT
metadata:
  author: dbos
  version: "1.0.0"
  organization: DBOS
  date: February 2026
  abstract: >
    Reference for the DBOS CLI across Python, TypeScript, and Go. Covers
    database migrations, project initialization, workflow management, local
    Postgres setup, and configuration.
---

# DBOS CLI Reference

Cross-language reference for the DBOS command-line interface. Commands vary by language SDK.

## When to Apply

Reference these guidelines when:
- Running `dbos migrate` to set up system tables or apply schema migrations
- Initializing a new DBOS project with `dbos init`
- Managing workflows from the command line (list, cancel, resume, fork)
- Starting a local Postgres instance for development
- Configuring `dbos-config.yaml` for migrations, rollbacks, or runtime
- Setting up database roles with least-privilege access
- Resetting the DBOS system database

## Rule Categories by Priority

| Priority | Category | Impact | Prefix |
|----------|----------|--------|--------|
| 1 | Migration | CRITICAL | `migrate-` |
| 2 | Configuration | HIGH | `config-` |
| 3 | Initialization | MEDIUM | `init-` |
| 4 | Workflow Management | MEDIUM | `workflow-` |
| 5 | Local Development | LOW-MEDIUM | `local-` |

## Critical Rules

### CLI Installation

**Python:**
```bash
pip install dbos
```

**TypeScript:**
```bash
npm install @dbos-inc/dbos-sdk@latest
```

**Go:**
```bash
go install github.com/dbos-inc/dbos-transact-golang/cmd/dbos@latest
```

### Database Migration

`dbos migrate` creates DBOS system tables and runs user-defined migration commands. This is the most critical CLI operation — DBOS cannot function without its system tables.

**Python:**
```bash
dbos migrate
dbos migrate --sys-db-url postgres://admin:pass@localhost/mydb
dbos migrate --app-role myapp_role
```

**TypeScript** (two separate commands):
```bash
# Create DBOS system tables
npx dbos schema postgres://admin:pass@localhost/mydb

# Run user-defined migrations from dbos-config.yaml
npx dbos migrate
```

**Go:**
```bash
dbos migrate
dbos migrate --db-url postgres://admin:pass@localhost/mydb
dbos migrate --app-role myapp_role
```

### Production Migration Pattern

In production, run `dbos migrate` with a privileged database user to create system tables and grant minimum permissions to the application role:

```bash
dbos migrate --sys-db-url postgres://admin:pass@host/db --app-role myapp_role
```

The application then runs as `myapp_role` with only the permissions it needs.

## How to Use

Read individual reference files for detailed explanations and examples:

```
references/migrate-system-tables.md
references/migrate-user-schema.md
references/config-database.md
```

## References

- https://docs.dbos.dev/python/reference/cli
- https://docs.dbos.dev/typescript/reference/configuration
- https://docs.dbos.dev/golang/programming-guide
