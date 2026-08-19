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
    implementation("dev.dbos:transact-spring-boot-starter:1.0.0")
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
`dbos.datasource.use-listen-notify`, `dbos.conductor.key`, `dbos.conductor.domain`, `dbos.executor-id`,
`dbos.enable-patching`, `dbos.listen-queues`, `dbos.scheduler-polling-interval`.

To adjust config programmatically without replacing it, declare a `DBOSConfigCustomizer` bean:

```java
@Bean
public DBOSConfigCustomizer myCustomizer() {
  return config -> config.withEnablePatching(true);
}
```

Settings with no `dbos.*` property yet — notably `withNotificationCoalesceInterval` and
`withDatabasePollingConcurrency` (see [lifecycle-config.md](lifecycle-config.md)) — are set the same way, through a
customizer.

Requirements and behavior:

- Beans with `@Workflow` or `@Step` methods must be singletons; prototype scope throws `IllegalStateException`
- When several beans of the same class exist, the `@Primary` bean uses the default instance name and the others are
  registered under their Spring bean name (see [workflow-instances.md](workflow-instances.md))
- Register schedules and database-backed queues after launch, e.g. from an
  `ApplicationListener<ContextRefreshedEvent>` bean

Reference: [Spring Boot Integration](https://docs.dbos.dev/java/tutorials/spring-boot-integration)
