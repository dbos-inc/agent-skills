---
title: Use Transactional Steps for Database Writes
impact: HIGH
impactDescription: Commits the database write and the DBOS checkpoint atomically, making writes exactly-once
tags: step, transaction, jdbc, jdbi, jooq, spring, exactly-once
---

## Use Transactional Steps for Database Writes

A regular step checkpoints its output after the body completes. If the process crashes between a database write and
the checkpoint, the step re-runs and writes twice. Step factories commit your database work and the step checkpoint
in the same transaction, so database writes become exactly-once.

**Incorrect (plain step around a database write):**

```java
@Workflow
public String processOrder(String orderId) throws Exception {
  // A crash after the INSERT but before the checkpoint duplicates the row on recovery
  return dbos.runStep(() -> {
    try (var conn = dataSource.getConnection();
         var stmt = conn.prepareStatement("INSERT INTO orders(id) VALUES (?)")) {
      stmt.setString(1, orderId);
      stmt.executeUpdate();
    }
    return orderId;
  }, "insertOrder");
}
```

**Correct (transactional step factory):**

```java
import dev.dbos.transact.txstep.JdbcStepFactory;

class OrderWorkflowImpl implements OrderWorkflow {
  private final JdbcStepFactory factory;

  OrderWorkflowImpl(DBOS dbos, DataSource dataSource) {
    // Construct before dbos.launch(); creates tx_step_outputs if missing
    this.factory = new JdbcStepFactory(dbos, dataSource);
  }

  @Override
  @Workflow
  public String processOrder(String orderId) throws Exception {
    return factory.txStep(conn -> {
      try (var stmt = conn.prepareStatement("INSERT INTO orders(id) VALUES (?)")) {
        stmt.setString(1, orderId);
        stmt.executeUpdate();
      }
      return orderId;
    }, "insertOrder");
  }
}
```

Available factories, by stack:

- Plain JDBC: `JdbcStepFactory` (core `dev.dbos:transact` module) — `txStep(conn -> ..., name)`
- JDBI 3: `JdbiStepFactory` (`dev.dbos:transact-jdbi-step-factory`) — `inStep(handle -> ..., name)` /
  `useStep(handle -> ..., name)` for the void variant
- jOOQ: `JooqStepFactory` (`dev.dbos:transact-jooq-step-factory`) — `txStepResult(trx -> ..., name)` /
  `txStep(trx -> ..., name)`
- Spring Boot: `@TransactionalStep` (`dev.dbos:transact-spring-txstep-starter`) on any Spring-managed method

```java
@Service
public class OrderStepService {
  @Autowired JdbcTemplate jdbc;

  @TransactionalStep(isolationLevel = Isolation.SERIALIZABLE)
  public Order saveOrder(Order order) {
    jdbc.update("INSERT INTO orders(id, item, qty) VALUES (?, ?, ?)",
        order.id(), order.item(), order.qty());
    return order;
  }
}
```

Rules:

- The callback receives an open connection/handle/configuration; it must not `commit`, `rollback`, or `close` it
- Construct factories before `dbos.launch()`; each verifies the datasource is PostgreSQL and creates the
  `tx_step_outputs` table in the configured schema, so the database user needs `CREATE TABLE` there
- Pass `StepFactoryOptions(name, IsolationLevel.SERIALIZABLE)` (or `JdbiStepOptions`) instead of a bare name to
  control isolation
- Steps retry automatically on PostgreSQL serialization failures (`40001`) and deadlocks (`40P01`)
- `@TransactionalStep` methods must be called through the Spring proxy; called outside a workflow, or from inside
  another step, they behave like plain `@Transactional` methods with no checkpoint

Reference: [Transactional Steps](https://docs.dbos.dev/java/tutorials/step-factory-tutorial)
