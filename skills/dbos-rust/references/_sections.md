# Section Definitions

This file defines the rule categories for DBOS Rust best practices. Rules are automatically assigned to sections based on their filename prefix.

---

## 1. Lifecycle (lifecycle)
**Impact:** CRITICAL
**Description:** DBOS configuration, required application versions, workflow registration order, launch and recovery, graceful shutdown, system database migrations, and Axum integration.

## 2. Workflow (workflow)
**Impact:** CRITICAL
**Description:** Workflow registration and WorkflowRef invocation, determinism rules, background and child workflows, multi-workflow coordination, timeouts, searchable attributes, introspection, and operational control.

## 3. Step (step)
**Impact:** HIGH
**Description:** Step execution with step and step_with, retries and per-attempt timeouts, deterministic racing with select_step!, and idempotent SQL writes inside steps.

## 4. Queue (queue)
**Impact:** HIGH
**Description:** Database-backed queue registration, worker and global concurrency limits, rate limiting, deduplication, partitioning, priority, delayed execution, selective listening, and live runtime reconfiguration.

## 5. Communication (comm)
**Impact:** MEDIUM
**Description:** Workflow events, durable FIFO messages, and append-only streams for communicating with and observing running workflows.

## 6. Pattern (pattern)
**Impact:** MEDIUM
**Description:** Core workflow patterns including workflow-ID idempotency, durable sleep, database-backed cron schedules, and burst debouncing.

## 7. Testing (test)
**Impact:** LOW-MEDIUM
**Description:** Unit testing step functions outside a live DBOS instance and isolating PostgreSQL integration tests.

## 8. Client (client)
**Impact:** MEDIUM
**Description:** Connecting external Rust services with dbos::Client to enqueue, retrieve, signal, and manage workflows without hosting an executor.

## 9. Advanced (advanced)
**Impact:** LOW
**Description:** Application version management for rolling and blue-green deploys, workflow patching, custom serde serializers, cross-language portable JSON interop, and shared system databases.
