---
title: Configure, Register, and Launch DBOS Properly
impact: CRITICAL
impactDescription: Application won't function without proper setup
tags: configuration, launch, setup, initialization, DBOSConfig
---

## Configure, Register, and Launch DBOS Properly

Every DBOS application creates a single `DBOS` instance from a `DBOSConfig`, registers its workflow classes with
`registerProxy`, and then calls `launch()`. Workflow recovery starts at launch, so every workflow class must be
registered before that point.

**Incorrect (workflows invoked without registration or launch):**

```java
public class App {
  public static void main(String[] args) {
    DBOS dbos = new DBOS(DBOSConfig.defaultsFromEnv("my-app"));

    // Calling the implementation directly — nothing is checkpointed,
    // and DBOS was never launched.
    new ExampleImpl(dbos).workflow("input");
  }
}
```

**Correct (configure, register, launch):**

```java
import dev.dbos.transact.DBOS;
import dev.dbos.transact.config.DBOSConfig;
import dev.dbos.transact.workflow.QueueOptions;
import dev.dbos.transact.workflow.Workflow;

public class App {
  public static void main(String[] args) {
    DBOSConfig config = DBOSConfig.defaultsFromEnv("my-app")
        .withAppVersion("0.1.0");

    // DBOS implements AutoCloseable; close() calls shutdown()
    try (DBOS dbos = new DBOS(config)) {
      ExampleImpl impl = new ExampleImpl(dbos);
      Example proxy = dbos.registerProxy(Example.class, impl);
      impl.setSelf(proxy); // so the class can call its own workflows/steps durably

      dbos.launch();

      // Database-backed queues are registered AFTER launch
      dbos.registerQueue("example-queue", QueueOptions.setWorkerConcurrency(5));

      proxy.workflow("input");
    }
  }
}
```

`DBOSConfig.defaultsFromEnv(appName)` reads connection settings from the environment:

- `DBOS_SYSTEM_JDBC_URL` — JDBC URL of the system database, e.g. `jdbc:postgresql://localhost:5432/mydb`
- `PGUSER` — PostgreSQL user (defaults to `postgres`)
- `PGPASSWORD` — password for that user

Use `DBOSConfig.defaults(appName)` plus `with` methods to configure explicitly:

- `withDatabaseUrl(String)` / `withDbUser(String)` / `withDbPassword(String)`: system database connection
- `withDataSource(DataSource)`: use an existing pooled `DataSource` instead of URL/credentials
- `withDatabaseSchema(String)`: schema for DBOS system tables (default `dbos`)
- `withAppVersion(String)`: code version for this application — set `"0.1.0"` in new applications
- `withMigrate(boolean)`: apply system database migrations on launch (default `true`)
- `withConductorKey(String)` / `withConductorDomain(String)`: connect to DBOS Conductor
- `withExecutorId(String)`: unique identifier for this process
- `withEnablePatching(boolean)`: enable workflow patching (default `false`)
- `withListenQueues(String...)`: only dequeue from these queues (default: all)
- `withSchedulerPollingInterval(Duration)`: how often scheduled workflows are polled (default 30s)
- `withUseListenNotify(boolean)`: use PostgreSQL `LISTEN`/`NOTIFY` for `recv`/`getEvent`/`readStream` (default
  `true`; automatically disabled on CockroachDB)
- `withNotificationCoalesceInterval(Duration)`: how often this process flushes the stream and workflow-event
  wake-ups it has queued for other processes (default 10ms, minimum 1ms). Raising it batches harder — fewer
  notifying commits, up to that much extra delivery latency
- `withDatabasePollingConcurrency(Integer)`: cap on concurrent DB-backed polling reads — the re-queries behind
  awaiting a result, `recv`, `getEvent`, and `readStream` (default: half the connection pool, minimum 1; a
  non-positive value removes the cap). The cap keeps a fan-out of waiters from holding every connection and
  starving enqueue/dequeue, status writes, recovery, and cancellation
- `withSerializer(DBOSSerializer)`: custom serializer, see [advanced-serialization.md](advanced-serialization.md)

To tune the system database connection pool, build your own pooled `DataSource` (DBOS uses HikariCP by default) and
pass it with `withDataSource(...)`. Size the pool for the workload, not just the polling cap: thousands of
concurrent waiters are fine on a small pool because each holds a connection only for its query, but the default cap
is derived from the pool size, so a bigger pool also raises how much of it polling may occupy.

Lifecycle rules:

- Register every workflow class (`registerProxy`) and alert handler before `launch()`
- Call `shutdown()` (or use try-with-resources) to release connections; in long-running servers, wire
  `launch()`/`shutdown()` into the server's own start/stop hooks
- Do not call workflows before `launch()` — methods that require a launched instance throw `IllegalStateException`

Register a handler for DBOS alerts before launch:

```java
dbos.registerAlertHandler((name, message, metadata) ->
    logger.warn("DBOS alert [{}]: {} {}", name, message, metadata));
```

Reference: [DBOS Lifecycle](https://docs.dbos.dev/java/reference/lifecycle)
