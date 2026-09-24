---
title: Initialize DBOSClient for External Access
impact: MEDIUM
impactDescription: Enables external applications to interact with DBOS workflows
tags: client, external, setup, DBOSClient, initialization
---

## Initialize DBOSClient for External Access

`DBOSClient` talks to a DBOS application's system database from outside that application — an API server, an admin
tool, or another service. It needs no registered workflows and no `launch()`, only database credentials.

**Incorrect (booting a full DBOS instance just to inspect workflows):**

```java
// Requires registering every workflow class and starts recovery,
// queue polling, and schedulers in a process that should only read state
DBOS dbos = new DBOS(config);
dbos.launch();
dbos.getWorkflowStatus(workflowId);
```

**Incorrect (never closing the client):**

```java
var client = new DBOSClient(url, user, password);
client.getWorkflowStatus(workflowId);
// Connection pool leaked - no close()!
```

**Correct (using DBOSClient with try-with-resources):**

```java
import dev.dbos.transact.DBOSClient;

try (var client = new DBOSClient(
    System.getenv("DBOS_SYSTEM_JDBC_URL"),
    System.getenv("PGUSER"),
    System.getenv("PGPASSWORD"))) {

  // Inspect and manage workflows
  Optional<WorkflowStatus> status = client.getWorkflowStatus(workflowId);
  WorkflowHandle<String, Exception> handle = client.retrieveWorkflow(workflowId);
  String result = handle.getResult();

  List<WorkflowStatus> failed = client.listWorkflows(
      new ListWorkflowsInput().withStatus(WorkflowState.ERROR).withLimit(20));
  List<StepInfo> steps = client.listWorkflowSteps(workflowId);

  client.cancelWorkflow(workflowId, true);       // cancel descendants too
  client.resumeWorkflow(workflowId);
  client.forkWorkflow(workflowId, 2, new ForkOptions());
  client.setWorkflowDelay(workflowId, Duration.ofMinutes(30));

  // Bulk variants
  client.cancelWorkflows(List.of("wf-1", "wf-2"));
  client.resumeWorkflows(List.of("wf-1", "wf-2"), "priority-queue");
  client.deleteWorkflows(List.of("wf-1"));

  // Communicate with running workflows
  client.send(workflowId, "approved", "approval", idempotencyKey);
  Optional<Object> event = client.getEvent(workflowId, "payment_id", Duration.ofSeconds(30));
  Iterator<Object> stream = client.readStream(workflowId, "results");
}
```

`DBOSClient` implements `AutoCloseable`; always `close()` it (or use try-with-resources) when done.

Constructors:

```java
new DBOSClient(String url, String user, String password)
new DBOSClient(String url, String user, String password, String schema)
new DBOSClient(String url, String user, String password, String schema, DBOSSerializer serializer)
new DBOSClient(String url, String user, String password, String schema, DBOSSerializer serializer,
    boolean useListenNotify)
new DBOSClient(String url, String user, String password, String schema, DBOSSerializer serializer,
    boolean useListenNotify, String applicationName)
new DBOSClient(DataSource dataSource)
new DBOSClient(DataSource dataSource, String schema)
new DBOSClient(DataSource dataSource, String schema, DBOSSerializer serializer)
new DBOSClient(DataSource dataSource, String schema, DBOSSerializer serializer, String applicationName)
new DBOSClient(DataSource dataSource, String schema, DBOSSerializer serializer, boolean useListenNotify)
new DBOSClient(DataSource dataSource, String schema, DBOSSerializer serializer, boolean useListenNotify,
    String applicationName)
```

Notes:

- `url` is the JDBC URL of the *system* database; `schema` defaults to `dbos`
- A client never migrates the system database. Construction checks that the schema is at least the minimum version
  this SDK needs and throws `IllegalStateException` if it is missing or too old — run the application (or
  `dbosctl sysdb migrate`) first
- `useListenNotify` defaults to `false` on the constructors that do not take it: it costs a dedicated connection and
  thread, and only `getEvent` and `readStream` benefit. Pass `true` for a client that waits on events or streams;
  leave it `false` if the database was migrated without notification triggers (`--no-listen-notify`)
- A `DBOSClient` must use the same serializer as the application whose workflows it touches
  ([advanced-serialization.md](advanced-serialization.md))
- The client also manages queues (`registerQueue`, `updateQueue`, `findQueue`, `listQueues`, `deleteQueue`); see
  [queue-management.md](queue-management.md)
- Unlike the Python and TypeScript clients, the Java client has no `waitFirst`, no `listQueuedWorkflows` (filter
  `listWorkflows` with `ListWorkflowsInput.withQueuesOnly(true)` instead), and no lazy connection
- `QueueConflictResolution.UPDATE_IF_LATEST_VERSION` is not available to clients since they have no application
  version — use `ALWAYS_UPDATE` (the client default) or `NEVER_UPDATE`
- To start work rather than inspect it, enqueue ([client-enqueue.md](client-enqueue.md))
- On a system database shared by several applications, pass `applicationName`: an unnamed client sees every
  application's rows but owns nothing it creates. `renameApplication(oldName, newName)` re-owns rows after a rename
  ([advanced-shared-database.md](advanced-shared-database.md))

## Schedule Management

Manage workflow schedules from outside the DBOS application. `WorkflowSchedule` names the workflow and its class as
strings instead of referencing a proxy ([pattern-scheduled.md](pattern-scheduled.md)):

```java
client.createSchedule(
    new WorkflowSchedule("my-task", "myPeriodicTask", "com.example.TasksImpl", "0 */5 * * * *")
        .withContext("my context"));

List<WorkflowSchedule> schedules = client.listSchedules(List.of(ScheduleStatus.ACTIVE), null, null);
Optional<WorkflowSchedule> schedule = client.getSchedule("my-task");
client.pauseSchedule("my-task");
client.resumeSchedule("my-task");
client.deleteSchedule("my-task");
client.applySchedules(List.of(dailyReport, hourlySync)); // Atomic batch create/update
client.backfillSchedule("my-task", start, end);
WorkflowHandle<Object, Exception> fired = client.triggerSchedule("my-task");
```

## Debouncing

```java
var debouncer = client.<String>debouncer("processInput")
    .withClassName("com.example.InputProcessorImpl");

void onUserInput(String userId, String userInput) {
  debouncer.debounce(userId, Duration.ofSeconds(60), userInput); // Wait 60s idle
}
```

See [pattern-debouncing.md](pattern-debouncing.md) for the options.

## Version Management

```java
List<VersionInfo> versions = client.listApplicationVersions();
VersionInfo latest = client.getLatestApplicationVersion();
client.setLatestApplicationVersion("1.0.0"); // Rollback
```

Reference: [DBOS Client](https://docs.dbos.dev/java/reference/client)
