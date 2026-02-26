---
title: Manage Local Postgres for Development
impact: LOW-MEDIUM
impactDescription: Simplifies local development setup with a single command
tags: postgres, docker, local, development, setup
---

## Manage Local Postgres for Development

The DBOS CLI can start and stop a local Postgres instance via Docker. Available in TypeScript and Go (Python uses SQLite by default for development).

**Incorrect (manual Docker setup with wrong settings):**

```bash
# Wrong image, missing pgvector, non-standard port
docker run -d -p 5433:5432 -e POSTGRES_PASSWORD=secret postgres:15
```

**Correct (use the built-in command):**

```bash
# Creates a correctly configured container with pgvector
dbos postgres start
```

### Start Local Postgres

```bash
# TypeScript
npx dbos postgres start

# Go
dbos postgres start
```

This creates a Docker container named `dbos-db` with:
- Port: 5432
- Database: `dbos`
- User: `postgres`
- Password: `dbos`
- Includes pgvector extension

### Stop Local Postgres

```bash
# TypeScript
npx dbos postgres stop

# Go
dbos postgres stop
```

### Python Alternative

Python's DBOS SDK uses SQLite by default for the system database during development, so no Postgres setup is needed. For Postgres, start a container manually:

```bash
docker run -d --name dbos-db \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD=dbos \
  -e POSTGRES_DB=dbos \
  pgvector/pgvector:pg17
```

### Connect to Local Instance

After starting, connect with:

```bash
export DBOS_SYSTEM_DATABASE_URL=postgres://postgres:dbos@localhost:5432/dbos
```

Reference: [Quickstart](https://docs.dbos.dev/quickstart)
