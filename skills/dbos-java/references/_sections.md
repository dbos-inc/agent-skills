# Section Definitions

This file defines the rule categories for DBOS Java best practices. Rules are automatically assigned to sections based on their filename prefix.

---

## 1. Lifecycle (lifecycle)
**Impact:** CRITICAL
**Description:** DBOS configuration, workflow registration, launch, and Spring Boot integration. Foundation for all DBOS applications.

## 2. Workflow (workflow)
**Impact:** CRITICAL
**Description:** Workflow registration and proxies, determinism requirements, background execution, timeouts, and workflow management.

## 3. Step (step)
**Impact:** HIGH
**Description:** Step creation with runStep and @Step, retry configuration, and transactional steps via step factories.

## 4. Queue (queue)
**Impact:** HIGH
**Description:** Queue registration and configuration, concurrency limits, rate limiting, partitioning, priority, and runtime management.

## 5. Communication (comm)
**Impact:** MEDIUM
**Description:** Workflow events, messages, and streaming for communicating with running workflows.

## 6. Pattern (pattern)
**Impact:** MEDIUM
**Description:** Common patterns including idempotency, scheduled workflows, durable sleep, and debouncing.

## 7. Testing (test)
**Impact:** LOW-MEDIUM
**Description:** Unit testing workflows with Mockito and integration testing with a real PostgreSQL database.

## 8. Client (client)
**Impact:** MEDIUM
**Description:** DBOSClient for interacting with DBOS from external applications.

## 9. Advanced (advanced)
**Impact:** LOW
**Description:** Workflow versioning, patching, serialization, cross-language interoperability, and Kotlin extensions.
