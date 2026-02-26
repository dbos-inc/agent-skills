---
title: Production Migration with Least-Privilege Roles
impact: HIGH
impactDescription: Prevents over-privileged database access in production environments
tags: migrate, production, app-role, permissions, security
---

## Production Migration with Least-Privilege Roles

In production, the application should not run as a database superuser. Use `--app-role` to create system tables as a privileged user and grant only the minimum permissions needed to an application role.

**Incorrect (application runs as admin):**

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

### Cross-Language Examples

**Python:**
```bash
dbos migrate -s postgres://admin:pass@host/db -r myapp_role
```

**TypeScript:**
```bash
npx dbos schema postgres://admin:pass@host/db --app-role myapp_role
npx dbos migrate  # Run user schema migrations separately
```

**Go:**
```bash
dbos migrate --db-url postgres://admin:pass@host/db --app-role myapp_role
```

### CI/CD Integration

A typical deployment pipeline:

```yaml
# GitHub Actions example
steps:
  - name: Run DBOS migrations
    env:
      ADMIN_DB_URL: ${{ secrets.ADMIN_DATABASE_URL }}
    run: dbos migrate --sys-db-url $ADMIN_DB_URL --app-role myapp_role

  - name: Start application
    env:
      DBOS_SYSTEM_DATABASE_URL: ${{ secrets.APP_DATABASE_URL }}
    run: dbos start
```

Reference: [Python CLI](https://docs.dbos.dev/python/reference/cli) | [Deploying to Cloud](https://docs.dbos.dev/production/dbos-cloud/deploying-to-cloud)
