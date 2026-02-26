# Section Definitions

This file defines the rule categories for DBOS CLI reference. Rules are automatically assigned to sections based on their filename prefix.

---

## 1. Migration (migrate)
**Impact:** CRITICAL
**Description:** Database migration commands for creating DBOS system tables, running user-defined schema migrations, and setting up production database permissions.

## 2. Configuration (config)
**Impact:** HIGH
**Description:** The dbos-config.yaml file structure, database connection settings, migration command definitions, and environment variable resolution.

## 3. Initialization (init)
**Impact:** MEDIUM
**Description:** Project scaffolding with dbos init, template selection, and adding DBOS to existing projects.

## 4. Workflow Management (workflow)
**Impact:** MEDIUM
**Description:** CLI commands for listing, inspecting, canceling, resuming, and forking workflows.

## 5. Local Development (local)
**Impact:** LOW-MEDIUM
**Description:** Local Postgres setup with Docker, development server startup, and database reset commands.
