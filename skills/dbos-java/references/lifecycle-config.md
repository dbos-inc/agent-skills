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
whichever process dequeues it runs it. The launch-time sweep is skipped when a Conductor key is configured, since
Conductor then decides which executors are gone and issues recovery itself. A workflow's
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
      dbos.registerQueue("example-queue", new QueueOptions().withWorkerConcurrency(5));

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
  accepts only 3-256 lowercase letters, digits, `-` and `_`: any other name fails launch when a Conductor key is set,
  and only logs a warning otherwise
- `withDatabaseUrl(String)` / `withDbUser(String)` / `withDbPassword(String)`: system database connection
- `withDataSource(DataSource)`: use an existing pooled `DataSource` instead of URL/credentials
- `withDatabaseSchema(String)`: schema for DBOS system tables (default `dbos`)
- `withAppVersion(String)`: code version for this application — set `"0.1.0"` in new applications
- `withMigrate(boolean)`: apply system database migrations on launch (default `true`). With `false`, launch only
  checks that the schema has every migration this release defines (125 for 1.3; a schema migrated further by a newer
  release is accepted) and throws `IllegalStateException` if it is missing or too old; migrate out-of-band with
  `dbosctl sysdb migrate` (`--app-role` grants the application's role access, `--print-migrations all|N` and
  `--print-user-role` print the SQL instead of running it, `--no-listen-notify` omits the notification triggers).
  There is no Java `dbos` CLI
- `withConductorKey(String)` / `withConductorDomain(String)`: connect to DBOS Conductor. Since 1.3 this requires the
  `dev.dbos:transact-enterprise` artifact on the class path, at exactly the same version as `dev.dbos:transact`
  (DBOS loads it through `ServiceLoader`; no code change). Without it, `launch()` throws `IllegalStateException`
  ("Connecting to DBOS Conductor requires dev.dbos:transact-enterprise on the class path..."); with a different
  version, launch fails naming both. Do not add it to applications that don't use Conductor. Since 1.3 the client
  also verifies Conductor's TLS certificate against the JVM's trust store: for a self-hosted Conductor behind a
  self-signed or private-CA certificate, pass a trust store with `-Djavax.net.ssl.trustStore=...` (and
  `-Djavax.net.ssl.trustStorePassword=...`), or the application keeps retrying and never connects
- `withConductorExecutorMetadata(Map<String, Object>)`: JSON-serializable metadata identifying this executor in the
  Conductor dashboard (region, instance type, ...)
- `withConductorMetadataOnlyMode(boolean)` (1.3+): send Conductor only workflow metadata, never inputs, outputs,
  errors, step outputs, events, messages or streams (default `false`)
- `withSystemDatabaseIdleTransactionTimeout(Duration)` (1.3+): `idle_in_transaction_session_timeout` for the pool
  DBOS creates (default 1 minute; zero or negative keeps the server's setting). Not applied to a supplied
  `DataSource` or when the JDBC URL already sets it
- `withExecutorId(String)`: unique identifier for this process
- `withEnablePatching(boolean)`: enable workflow patching (default `false`)
- `withListenQueues(String...)`: only dequeue from these queues (default: all)
- `withSchedulerPollingInterval(Duration)`: how often scheduled workflows are polled (default 30s,
  `DBOSConfig.DEFAULT_SCHEDULER_POLLING_INTERVAL`). Since 1.3 it must be at least 1ms and at most `Long.MAX_VALUE`
  ms; zero, negative, sub-millisecond or larger values throw `IllegalArgumentException` when the config is built
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

Upgrading a fleet: migrations apply to the whole system database as soon as the first node on a new release starts.
Upgrade every node to 1.1 before rolling out 1.2. 1.2 stores workflow inputs and outputs only in the
`workflow_input` / `workflow_output` tables, which 1.0 can't read, and 1.0 doesn't release the key of a workflow that
1.2 debounced. 1.1 and 1.2 can share a fleet and 1.2 can roll back to 1.1, but never run 1.0 alongside 1.2 or roll
1.2 back to 1.0.

Upgrading to 1.3: it migrates the system database to version 125. With `withMigrate(false)`, run
`dbosctl sysdb migrate` (dbosctl v0.12.0+) first. A 1.3 `DBOSClient` never migrates and throws
`IllegalStateException` against a schema below 125, so upgrade clients after a 1.3 executor has launched. Conductor
users add `dev.dbos:transact-enterprise` (above). A schedule without `cronTimezone` now fires in UTC rather than the
JVM's zone; set `withCronTimezone` to keep a local zone. Transactional step factories must be created before
`launch()` (1.3 throws `IllegalStateException` otherwise).

Mixing 1.2 and 1.3 executors during a rolling upgrade:

- The computed application version includes the DBOS version, so unless `withAppVersion` pins it, 1.3 executors
  don't recover workflows 1.2 left `PENDING` or dequeue ones 1.2 enqueued (they keep the old version, and resuming
  doesn't change it). Keep 1.2's `withAppVersion` value, or set 1.3's to the version 1.2 computed, or fork that work
  with `new ForkOptions().withApplicationVersion(...)` naming the 1.3 version
- Don't pause a queue (`concurrency` or `workerConcurrency` 0) until every executor runs 1.3: 1.2 can't read such a
  queue, so its queue discovery fails on every poll and it keeps dequeuing with the limits it last read
- 1.2 doesn't fence writes on the owning execution, so a 1.2 executor can still write a step for a stale run; the
  1.3 owner then fails the workflow with `DBOSStepNondeterminismException`
- On hosts not in UTC, a schedule without `cronTimezone` also gets new tick workflow IDs (1.2 built them in the host's
  zone, 1.3 in UTC), even if its firing times don't move. While 1.2 and 1.3 run together every tick runs twice, and a
  1.3 backfill over ticks 1.2 fired runs them again. On the first start after the upgrade, automatic backfill reads
  the schedule in UTC from a `lastFiredAt` recorded in the host's zone and may enqueue a tick that was never due. For
  such apps, don't run 1.2 and 1.3 together and don't backfill across the upgrade

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
