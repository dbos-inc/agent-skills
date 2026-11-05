# Section Definitions

This file defines the rule categories for DBOS Rust best practices. Rules are automatically assigned to sections based on their filename prefix.

---

## 1. Lifecycle (lifecycle)
**Impact:** CRITICAL
**Description:** DBOS configuration, registration, launch and shutdown, the required application version, and choosing an async runtime with Cargo features. Foundation for all DBOS applications.

## 2. Workflow (workflow)
**Impact:** CRITICAL
**Description:** Workflow creation, determinism, durable concurrency (`select_step!` and joins instead of races, timeouts, spawns and completion-order streams), error types, background execution, timeouts, attributes, and management (cancel, resume, fork, rewind).

## 3. Step (step)
**Impact:** HIGH
**Description:** Step creation, retries, timeouts and cancellation, running steps concurrently, and when to use steps.

## 4. Queue (queue)
**Impact:** HIGH
**Description:** Queue creation, concurrency limits, rate limiting, partitioning, deduplication, priority, and which queues a process listens to.

## 5. Communication (comm)
**Impact:** MEDIUM
**Description:** Workflow events, messages, and streams for communicating with workflows.

## 6. Pattern (pattern)
**Impact:** MEDIUM
**Description:** Common patterns including idempotency, scheduled workflows, debouncing, delayed execution and durable sleep.

## 7. Testing (test)
**Impact:** LOW-MEDIUM
**Description:** Unit-testing functions built from steps, and integration tests against Postgres.

## 8. Client (client)
**Impact:** MEDIUM
**Description:** The DBOS Client for interacting with DBOS from external applications.

## 9. Advanced (advanced)
**Impact:** LOW
**Description:** Application versions, patching, versioning, custom serialization, portable workflows across languages, and sharing a system database.
