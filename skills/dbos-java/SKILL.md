---
name: dbos-java
description: DBOS Java SDK for building reliable, fault-tolerant applications with durable workflows. Use this skill when writing Java or Kotlin code with DBOS, creating workflows and steps, using queues, integrating DBOS with Spring Boot, using DBOSClient from external applications, or building JVM applications that need to be resilient to failures.
license: MIT
metadata:
  author: dbos
  version: "1.0.0"
  organization: DBOS
  date: July 2026
  abstract: Comprehensive guide for building fault-tolerant Java applications with DBOS. Covers workflow registration and proxies, steps, queues, communication patterns, Spring Boot integration, and best practices for durable execution.
---

# DBOS Java Best Practices

Guide for building reliable, fault-tolerant Java applications with DBOS durable workflows.

## When to Apply

Reference these guidelines when:
- Adding DBOS to existing Java or Kotlin code
- Creating workflows and steps
- Using queues for concurrency control
- Implementing workflow communication (events, messages, streams)
- Configuring and launching DBOS applications, including with Spring Boot
- Using DBOSClient from external applications
- Testing DBOS applications

## Rule Categories by Priority

| Priority | Category | Impact | Prefix |
|----------|----------|--------|--------|
| 1 | Lifecycle | CRITICAL | `lifecycle-` |
| 2 | Workflow | CRITICAL | `workflow-` |
| 3 | Step | HIGH | `step-` |
| 4 | Queue | HIGH | `queue-` |
| 5 | Communication | MEDIUM | `comm-` |
| 6 | Pattern | MEDIUM | `pattern-` |
| 7 | Testing | LOW-MEDIUM | `test-` |
| 8 | Client | MEDIUM | `client-` |
| 9 | Advanced | LOW | `advanced-` |

## Critical Rules

### Installation

Add the DBOS dependency (Java 17+, Gradle 8+ recommended):

```kotlin
dependencies {
    implementation("dev.dbos:transact:1.0.0")
    implementation("org.slf4j:slf4j-simple:2.0.17") // to see DBOS log messages
}
```

DBOS requires a PostgreSQL-compatible database. Connection settings are typically supplied through
`DBOS_SYSTEM_JDBC_URL`, `PGUSER`, and `PGPASSWORD`.

### DBOS Configuration and Launch

A DBOS application MUST create a `DBOS` instance, register its workflow classes, then launch:

```java
import dev.dbos.transact.DBOS;
import dev.dbos.transact.config.DBOSConfig;
import dev.dbos.transact.workflow.Workflow;

interface Example {
  String workflow(String input);
}

class ExampleImpl implements Example {
  private final DBOS dbos;

  ExampleImpl(DBOS dbos) {
    this.dbos = dbos;
  }

  @Override
  @Workflow
  public String workflow(String input) {
    return dbos.runStep(() -> fetchData(input), "fetchData");
  }

  private String fetchData(String input) { /* external call */ return input; }
}

public class App {
  public static void main(String[] args) {
    DBOSConfig config = DBOSConfig.defaultsFromEnv("my-app")
        .withAppVersion("0.1.0");
    DBOS dbos = new DBOS(config);

    Example proxy = dbos.registerProxy(Example.class, new ExampleImpl(dbos));

    dbos.launch();

    proxy.workflow("input"); // durable: called through the proxy
  }
}
```

When creating a new application, set `withAppVersion("0.1.0")`. If omitted, DBOS derives an opaque hash from
workflow source code. When editing an existing application, leave its configured version alone — changing it is a
deployment decision (see `references/advanced-versioning.md`).

### Workflow and Step Structure

Workflows are ordinary methods annotated `@Workflow` on a class registered with `registerProxy`. Any operation that
is complex, non-deterministic, or calls an external service must run as a step, either through `dbos.runStep` or as
a `@Step`-annotated method invoked through the proxy:

```java
@Workflow
public String myWorkflow(String url) throws Exception {
  // Inline lambda step — use for one-off work
  String body = dbos.runStep(() -> httpGet(url), "httpGet");
  // Reusable step declared with @Step — MUST be invoked through the proxy (self)
  return self.parse(body);
}

@Step(maxAttempts = 3)
public String parse(String body) { /* ... */ }
```

### Key Constraints

- Workflows and `@Step` methods are ONLY durable when invoked through the proxy returned by `registerProxy`
  (or, with Spring Boot, through an injected self-reference). Calls via `this` bypass DBOS.
- All workflow classes MUST be registered before `launch()`; database-backed queues are registered after `launch()`
- Do NOT start, enqueue, or retrieve workflows from within a step, and do NOT call `send`, `recv`, `setEvent`,
  or `getEvent` from a step
- Workflows MUST be deterministic — random numbers, current time, I/O, and threading go in steps
- Do NOT mutate state outside the workflow's own scope from workflows or steps
- Workflow arguments and step/workflow return values MUST be JSON-serializable by Jackson
- Use `dbos.sleep`, not `Thread.sleep`, inside workflows

## How to Use

Read individual rule files for detailed explanations and examples:

```
references/lifecycle-config.md
references/workflow-registration.md
references/queue-concurrency.md
```

## References

- https://docs.dbos.dev/
- https://github.com/dbos-inc/dbos-transact-java
