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

**Correct (using DBOSClient):**

```java
import dev.dbos.transact.DBOSClient;

var client = new DBOSClient(
    System.getenv("DBOS_SYSTEM_JDBC_URL"),
    System.getenv("PGUSER"),
    System.getenv("PGPASSWORD"));

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

// Communicate with running workflows
client.send(workflowId, "approved", "approval", idempotencyKey);
Optional<Object> event = client.getEvent(workflowId, "payment_id", Duration.ofSeconds(30));
Iterator<Object> stream = client.readStream(workflowId, "results");
```

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
- The client also manages queues (`registerQueue`, `updateQueue`, `findQueue`, `listQueues`, `deleteQueue`),
  schedules (`applySchedules`, `createSchedule`, `pauseSchedule`, `triggerSchedule`, `backfillSchedule`, ...), and
  application versions (`listApplicationVersions`, `setLatestApplicationVersion`)
- `QueueConflictResolution.UPDATE_IF_LATEST_VERSION` is not available to clients since they have no application
  version — use `ALWAYS_UPDATE` (the client default) or `NEVER_UPDATE`
- To start work rather than inspect it, enqueue ([client-enqueue.md](client-enqueue.md))

Reference: [DBOS Client](https://docs.dbos.dev/java/reference/client)
