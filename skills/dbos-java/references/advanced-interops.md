---
title: Interoperate with Python and TypeScript Applications
impact: LOW
impactDescription: Enables workflows, events, and messages to cross language boundaries
tags: advanced, interop, portable, cross-language, serialization
---

## Interoperate with Python and TypeScript Applications

DBOS applications in different languages can share one system database. Interop requires two things: a portable
serialization format, and stable workflow and class names that other languages can address.

**Incorrect (default Java serialization across languages):**

```java
// java_jackson payloads and a fully-qualified Java class name;
// a Python or TypeScript worker cannot address or decode this workflow
@Workflow
public String processOrder(String orderId, long quantity) { /* ... */ }
```

**Correct (portable serialization and a stable class name):**

```java
import dev.dbos.transact.workflow.SerializationStrategy;
import dev.dbos.transact.workflow.WorkflowClassName;

@WorkflowClassName("OrderService") // registered as processOrder/OrderService
public class OrderServiceImpl implements OrderService {

  @Override
  @Workflow(serializationStrategy = SerializationStrategy.PORTABLE)
  public String processOrder(String orderId, long quantity) {
    return "order:" + orderId + " qty:" + quantity;
  }
}
```

Enqueue a workflow whose implementation lives in another language, using portable JSON arguments:

```java
var options = new DBOSClient.EnqueueOptions("process_order", "OrderService", "order-queue")
    .withSerialization(SerializationStrategy.PORTABLE);

// positional args, then named args (for languages that support them, e.g. Python kwargs)
var handle = client.enqueuePortableWorkflow(
    options, new Object[] {"order-123", 5}, Map.of());
```

Cross-language notes:

- `SerializationStrategy.PORTABLE` (`portable_json`) is also accepted by `send`, `setEvent`, and `writeStream`, so
  messages, events, and streams can be consumed by any language
- Java coerces portable JSON arguments to the method's parameter types automatically: JSON integers widen to
  `long`, decimals to `double`, ISO-8601 strings parse to `Instant` or `OffsetDateTime`, arrays to `List`, objects
  to `Map<String, Object>`. Coercion failure marks the workflow `ERROR` with a descriptive message.
- Use `@WorkflowClassName` so other languages address a short, stable name instead of a Java package path, and set
  `@Workflow(name = "...")` when the workflow name must match another language's naming convention
- Queues, schedules, and workflow management operate on the same system database regardless of language, so a Java
  `DBOSClient` can inspect, cancel, or resume workflows owned by a Python or TypeScript application

Reference: [Serialization Strategy](https://docs.dbos.dev/java/reference/methods#serialization-strategy)
