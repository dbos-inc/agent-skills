---
title: Reset the DBOS System Database
impact: LOW-MEDIUM
impactDescription: Clears workflow metadata for a clean development restart
tags: reset, system-database, development, cleanup
---

## Reset the DBOS System Database

`dbos reset` deletes all DBOS system tables and workflow metadata. Use this during development to start fresh.

**Incorrect (dropping the database manually):**

```bash
# Drops everything including application data
psql -c "DROP DATABASE mydb;"
```

**Correct (reset only DBOS system tables):**

```bash
# Removes only DBOS workflow metadata, preserves application data
dbos reset
```

**Python:**
```bash
dbos reset
```

**TypeScript:**
```bash
npx dbos reset
```

**Go:**
```bash
dbos reset
```

**Warning:** This permanently deletes all workflow state, step results, and event history. Never run in production unless you intend to lose all workflow recovery data.

### When to Use Reset

- After changing workflow structure during development
- When system tables are in a corrupted state
- To clear out test data between development cycles

### After Reset

Re-run migrations to recreate system tables:

```bash
dbos migrate
```

Reference: [Python CLI](https://docs.dbos.dev/python/reference/cli) | [TypeScript CLI](https://docs.dbos.dev/typescript/reference/cli) | [Go CLI](https://docs.dbos.dev/golang/reference/cli)
