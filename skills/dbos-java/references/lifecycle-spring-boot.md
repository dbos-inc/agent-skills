---
title: Integrate DBOS with Spring Boot Using the Starter
impact: CRITICAL
impactDescription: Auto-configures DBOS lifecycle and workflow registration in Spring applications
tags: spring, spring-boot, starter, autoconfiguration, lifecycle
---

## Integrate DBOS with Spring Boot Using the Starter

The `transact-spring-boot-starter` auto-configures `DBOSConfig`, the `DBOS` bean, workflow registration, and the
launch/shutdown lifecycle. With the starter you annotate methods on Spring singletons directly — no interface and no
`registerProxy` call. Because Spring AOP only intercepts calls that pass through the Spring proxy, self-calls must
go through an injected self-reference.

**Incorrect (manual lifecycle and `this` self-calls):**

```java
@Service
public class OrderService {
  @Autowired DBOS dbos;

  @Workflow
  public String processOrder(String orderId) {
    // `this.chargeCard(...)` bypasses the Spring proxy: no checkpoint is recorded
    return this.chargeCard(orderId);
  }

  @Step
  public String chargeCard(String orderId) { /* ... */ return "charged"; }
}
```

**Correct (starter auto-configuration with a self-reference):**

```kotlin
// build.gradle.kts
dependencies {
    implementation("dev.dbos:transact-spring-boot-starter:1.3.0")
    // Only if the app connects to DBOS Conductor (dbos.conductor.key), 1.3+; same version as the starter
    // implementation("dev.dbos:transact-enterprise:1.3.0")
}
```

```yaml
# application.yaml
dbos:
  application:
    name: "my-app"
    version: "0.1.0"
  datasource:
    url: "jdbc:postgresql://localhost:5432/my_app_db"
    username: "postgres"
    password: "${PGPASSWORD}"
```

```java
@Service
public class OrderService {

  private final DBOS dbos;
  private OrderService self;

  public OrderService(DBOS dbos) {
    this.dbos = dbos;
  }

  @Autowired
  @Lazy
  public void setSelf(OrderService self) {
    this.self = self;
  }

  @Workflow
  public String processOrder(String orderId) {
    String result = self.chargeCard(orderId); // intercepted — durable
    self.sendConfirmation(orderId, result);
    return result;
  }

  @Step
  public String chargeCard(String orderId) { /* ... */ return "charged"; }

  @Step
  public void sendConfirmation(String orderId, String result) { /* ... */ }
}
```

Auto-configured beans (all `@ConditionalOnMissingBean`, so any of them can be replaced):

- `DBOSConfig` — built from `dbos.*` properties, falling back to `spring.application.name` and `spring.datasource.*`
- `DBOS` — the DBOS instance, injectable anywhere
- `DBOSLifecycle` — `SmartLifecycle` that calls `launch()` on context start and `shutdown()` on stop
- `DBOSAspect` — intercepts `@Workflow` and `@Step` calls on Spring beans
- `DBOSWorkflowRegistrar` — registers every singleton bean containing `@Workflow` methods

Key configuration properties: `dbos.application.name`, `dbos.application.version`, `dbos.datasource.url`,
`dbos.datasource.username`, `dbos.datasource.password`, `dbos.datasource.schema`, `dbos.datasource.migrate`,
`dbos.datasource.use-listen-notify`, `dbos.conductor.key`, `dbos.conductor.domain`,
`dbos.conductor.metadata-only-mode` (1.3+), `dbos.executor-id`, `dbos.enable-patching`, `dbos.listen-queues`,
`dbos.scheduler-polling-interval` (at least 1ms since 1.3). Setting `dbos.conductor.key` requires
`dev.dbos:transact-enterprise` at the same version as the starter (1.3+); launch fails without it.

To adjust config programmatically without replacing it, declare a `DBOSConfigCustomizer` bean:

```java
@Bean
public DBOSConfigCustomizer myCustomizer() {
  return config -> config.withEnablePatching(true);
}
```

Settings with no `dbos.*` property yet — notably `withNotificationCoalesceInterval`,
`withDatabasePollingConcurrency` and `withSystemDatabaseIdleTransactionTimeout` (see
[lifecycle-config.md](lifecycle-config.md)) — are set the same way, through a customizer.

Without `dbos.datasource.url`, the starter hands DBOS the application's `DataSource` bean (`withDataSource`), so DBOS
creates no pool and its 1-minute idle-in-transaction timeout (1.3+) is not applied. To keep a stalled session from
holding a workflow's row lock, set `idle_in_transaction_session_timeout` on that pool yourself, e.g.
`spring.datasource.hikari.connection-init-sql=SET idle_in_transaction_session_timeout = '1min'` (this also bounds the
application's own transactions on that pool).

Requirements and behavior:

- Beans with `@Workflow` or `@Step` methods must be singletons; prototype scope throws `IllegalStateException`
- When several beans of the same class exist, the `@Primary` bean uses the default instance name and the others are
  registered under their Spring bean name (see [workflow-instances.md](workflow-instances.md))
- Register schedules and database-backed queues after launch, e.g. from an
  `ApplicationListener<ContextRefreshedEvent>` bean

Reference: [Spring Boot Integration](https://docs.dbos.dev/java/tutorials/spring-boot-integration)
