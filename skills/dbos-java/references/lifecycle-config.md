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

At launch, the process re-enqueues its own `PENDING` workflows from the current application version rather than
running them in-process: each goes back onto its own queue (or the internal queue if it was started directly), and
whichever process dequeues it runs it. The launch-time sweep is skipped when a Conductor key is configured or on DBOS
Cloud, where Conductor decides which executors are gone and issues recovery itself. A workflow's
`maxRecoveryAttempts` budget is counted at each dequeue, so a recovered workflow spends an attempt when it is
dispatched from the queue, not when it is re-enqueued.

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

For scheduled-only applications (no HTTP server), keep the process alive after launch instead of closing DBOS:

```java
dbos.launch();
dbos.applySchedules(
    new WorkflowSchedule("my-task", "scheduledTask", "com.example.TasksImpl", "0 * * * * *"));
Thread.currentThread().join(); // Block forever
```

`DBOSConfig.defaultsFromEnv(appName)` reads connection settings from the environment:

- `DBOS_SYSTEM_JDBC_URL` — JDBC URL of the system database, e.g. `jdbc:postgresql://localhost:5432/mydb`
- `PGUSER` — PostgreSQL user (defaults to `postgres`)
- `PGPASSWORD` — password for that user

Use `DBOSConfig.defaults(appName)` plus `with` methods to configure explicitly:

- `withAppName(String)`: the application name (required; also the argument to `defaults`/`defaultsFromEnv`).
  Applications sharing a system database must each have a distinct name — it identifies which application owns each
  workflow, queue, schedule, and version ([advanced-shared-database.md](advanced-shared-database.md)). DBOS Conductor
  accepts only 3-256 lowercase letters, digits, `-` and `_`: any other name fails launch when a Conductor key is set
  or on DBOS Cloud, and only logs a warning otherwise
- `withDatabaseUrl(String)` / `withDbUser(String)` / `withDbPassword(String)`: system database connection
- `withDataSource(DataSource)`: use an existing pooled `DataSource` instead of URL/credentials
- `withDatabaseSchema(String)`: schema for DBOS system tables (default `dbos`)
- `withAppVersion(String)`: code version for this application — set `"0.1.0"` in new applications
- `withMigrate(boolean)`: apply system database migrations on launch (default `true`). With `false`, launch only
  checks that the schema is at least the minimum version this SDK needs and throws `IllegalStateException` if it is
  missing or too old; migrate out-of-band with `dbosctl sysdb migrate` (`--app-role` grants the application's role
  access, `--print-migrations all|N` and `--print-user-role` print the SQL instead of running it,
  `--no-listen-notify` omits the notification triggers). There is no Java `dbos` CLI
- `withConductorKey(String)` / `withConductorDomain(String)`: connect to DBOS Conductor
- `withConductorExecutorMetadata(Map<String, Object>)`: JSON-serializable metadata identifying this executor in the
  Conductor dashboard (region, instance type, ...)
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

Connection poolers: when connecting through a **transaction-mode** pooler (PgBouncer in transaction mode, Supabase
Supavisor, Neon, PlanetScale), set `withUseListenNotify(false)`. `LISTEN` is connection-scoped, so a pooler that
hands the server connection back after each transaction orphans the registration and notifications are silently
dropped — `recv` and `getEvent` then fall back to re-checking only once a minute. With it off, DBOS polls the system
database every second instead.
Session-mode poolers keep a 1:1 connection mapping and work with `LISTEN`/`NOTIFY`.

Lifecycle rules:

- Register every workflow class (`registerProxy`) and alert handler before `launch()`
- Call `shutdown()` (or use try-with-resources) to release connections; in long-running servers, wire
  `launch()`/`shutdown()` into the server's own start/stop hooks
- Do not call workflows before `launch()` — methods that require a launched instance throw `IllegalStateException`
- System database failures DBOS will not retry surface as `DBOSSystemDatabaseException` (a `RuntimeException`):
  `sqlState()` returns the SQLSTATE and `getCause()` the database's own exception (`databaseException()` is
  deprecated). Connectivity failures arrive only after retries are exhausted; non-retryable ones (constraint
  violation, missing relation) arrive immediately

Register a handler for DBOS alerts before launch:

```java
dbos.registerAlertHandler((name, message, metadata) ->
    logger.warn("DBOS alert [{}]: {} {}", name, message, metadata));
```

Reference: [DBOS Lifecycle](https://docs.dbos.dev/java/reference/lifecycle)
