---
title: Initialize a New DBOS Project
impact: MEDIUM
impactDescription: Correct project scaffolding avoids configuration mistakes
tags: init, scaffolding, template, new-project
---

## Initialize a New DBOS Project

Each language SDK provides a project initialization command.

**Incorrect (manual setup without dbos init):**

```bash
# Creating files manually — easy to miss required configuration
mkdir my-app && cd my-app
pip install dbos
# No dbos-config.yaml, no template structure
```

**Correct (use dbos init for proper scaffolding):**

```bash
dbos init my-app
cd my-app
# Project has dbos-config.yaml, template code, and correct structure
```

### Python

```bash
# Create from default template
dbos init my-app

# Choose a specific template
dbos init my-app --template dbos-app-starter
dbos init my-app --template dbos-toolbox
dbos init my-app --template dbos-cron-starter
dbos init my-app --template dbos-db-starter

# Add config to existing project (creates dbos-config.yaml only)
dbos init --config
```

### TypeScript

```bash
# Interactive project creation
npx @dbos-inc/create@latest

# With template
npx @dbos-inc/create@latest --template dbos-node-starter
```

### Go

```bash
# Create from template
dbos init my-app
```

### Adding DBOS to an Existing Project

Instead of scaffolding, you can add DBOS to an existing codebase:

**Python:**
```bash
pip install dbos
dbos init --config  # Only creates dbos-config.yaml
```

**TypeScript:**
```bash
npm install @dbos-inc/dbos-sdk@latest
```

**Go:**
```bash
go get github.com/dbos-inc/dbos-transact-golang/dbos@latest
go install github.com/dbos-inc/dbos-transact-golang/cmd/dbos@latest
```

Then create `dbos-config.yaml` manually or with `dbos init --config`.

Reference: [Quickstart](https://docs.dbos.dev/quickstart) | [Python CLI](https://docs.dbos.dev/python/reference/cli) | [TypeScript CLI](https://docs.dbos.dev/typescript/reference/cli) | [Go Integrating DBOS](https://docs.dbos.dev/golang/integrating-dbos)
