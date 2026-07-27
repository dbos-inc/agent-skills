---
title: Keep Workflow Data Serializable
impact: LOW
impactDescription: Non-serializable data breaks checkpointing and recovery
tags: advanced, serialization, jackson, DBOSSerializer, types
---

## Keep Workflow Data Serializable

DBOS stores workflow arguments and workflow/step return values in PostgreSQL, serialized with Jackson by default.
Types that Jackson cannot round-trip either fail to serialize or come back as a different type after recovery,
breaking downstream code.

**Incorrect (passing non-serializable objects through workflow boundaries):**

```java
@Workflow
public void processOrder(Connection conn, Consumer<String> callback) {
  // Connections, streams, lambdas, and thread pools cannot be serialized;
  // this workflow cannot be checkpointed or recovered
}
```

**Correct (plain data in, plain data out):**

```java
// Records and POJOs Jackson can round-trip
public record Order(String id, String item, int quantity) {}

@Workflow
public Order processOrder(String orderId) {
  // Resources are created inside steps, not passed as arguments
  return dbos.runStep(() -> repository.load(orderId), "loadOrder");
}
```

Guidelines:

- Use records, POJOs with getters, collections, and standard value types (`String`, numbers, `Instant`, `UUID`)
- Add Jackson annotations where the default mapping is wrong; for example, a field that should be serialized but not
  deserialized: `@JsonProperty(access = JsonProperty.Access.READ_ONLY)`
- Keep resources (connections, HTTP clients, file handles) inside steps; pass identifiers across boundaries
- A step that returns a polymorphic type needs Jackson type information (`@JsonTypeInfo`), or it will deserialize to
  the declared base type

To change the format globally, implement `DBOSSerializer` and pass it to the configuration:

```java
import dev.dbos.transact.json.DBOSSerializer;

public class MyCustomSerializer implements DBOSSerializer {
  @Override public String name() { return "my_custom"; }
  @Override public String serialize(Object value) { /* ... */ }
  @Override public Object deserialize(String text) { /* ... */ }
  @Override public String serializeThrowable(Throwable throwable) { /* ... */ }
  @Override public Throwable deserializeThrowable(String text) { /* ... */ }
}

var config = DBOSConfig.defaultsFromEnv("my-app")
    .withAppVersion("0.1.0")
    .withSerializer(new MyCustomSerializer());
```

`name()` is stored alongside every value so DBOS knows how to read it back. Any `DBOSClient` that touches this
application's workflows must be constructed with the same serializer:

```java
var client = new DBOSClient(dbUrl, dbUser, dbPassword, null, new MyCustomSerializer());
```

Per-call strategies: `send`, `setEvent`, `writeStream`, and `EnqueueOptions` accept a `SerializationStrategy` —
`DEFAULT` (the configured serializer, `java_jackson` unless the workflow is portable), `NATIVE` (always Jackson), or
`PORTABLE` (cross-language JSON, see [advanced-interops.md](advanced-interops.md)).

Reference: [Custom Serialization](https://docs.dbos.dev/java/reference/lifecycle#custom-serialization)
