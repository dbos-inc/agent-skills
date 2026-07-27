---
title: Register Workflow Classes and Invoke Them Through the Proxy
impact: CRITICAL
impactDescription: Workflows called outside the proxy are not durable and cannot recover
tags: workflow, registration, proxy, registerProxy, annotation
---

## Register Workflow Classes and Invoke Them Through the Proxy

In plain Java, DBOS adds durability with a dynamic proxy. Define an interface, annotate the implementation's methods
with `@Workflow`, and register the implementation with `registerProxy`. Only calls made through the returned proxy
are checkpointed — a call on the implementation object itself runs as an ordinary method.

**Incorrect (calling the implementation instance):**

```java
class ExampleImpl implements Example {
  private final DBOS dbos;

  ExampleImpl(DBOS dbos) { this.dbos = dbos; }

  @Override
  @Workflow
  public String workflow(String input) {
    return dbos.runStep(() -> process(input), "process");
  }
}

ExampleImpl impl = new ExampleImpl(dbos);
dbos.registerProxy(Example.class, impl);
dbos.launch();

impl.workflow("input"); // NOT durable — bypasses the proxy, no workflow record
```

**Correct (invoking the registered proxy, including for self-calls):**

```java
interface Example {
  String workflow(String input);
  String childWorkflow(String task);
}

class ExampleImpl implements Example {
  private final DBOS dbos;
  private Example self; // proxy, injected after registration

  ExampleImpl(DBOS dbos) { this.dbos = dbos; }

  void setSelf(Example self) { this.self = self; }

  @Override
  @Workflow
  public String workflow(String input) throws Exception {
    String processed = dbos.runStep(() -> process(input), "process");
    // Child workflows are started through the proxy too
    var handle = dbos.startWorkflow(() -> self.childWorkflow(processed));
    return handle.getResult();
  }

  @Override
  @Workflow
  public String childWorkflow(String task) { return "done:" + task; }

  private String process(String input) { return input.trim(); }
}

ExampleImpl impl = new ExampleImpl(dbos);
Example proxy = dbos.registerProxy(Example.class, impl);
impl.setSelf(proxy);
dbos.launch();

proxy.workflow("input"); // durable
```

`@Workflow` parameters:

- `name`: workflow name, unique within the class (defaults to the method name). Changing it changes the identity
  used to recover existing workflows.
- `maxRecoveryAttempts`: dead-letter limit. After this many attempts, the workflow's status becomes
  `MAX_RECOVERY_ATTEMPTS_EXCEEDED` and it is no longer executed.
- `serializationStrategy`: set to `SerializationStrategy.PORTABLE` for cross-language invocation, see
  [advanced-interops.md](advanced-interops.md).

Registration rules:

- `registerProxy(Class<T> interfaceClass, T implementation)` must be called before `launch()`; the first argument
  must be an interface, and the implementation must declare at least one `@Workflow` or `@Step` method
- Pass a third argument (`instanceName`) when registering multiple instances of the same class, see
  [workflow-instances.md](workflow-instances.md)
- Annotate the implementation class with `@WorkflowClassName("ShortName")` to give workflows a stable, refactoring-
  proof class name
- With `transact-spring-boot-starter` no interface or `registerProxy` call is needed — annotate methods on Spring
  singletons and inject a self-reference instead, see [lifecycle-spring-boot.md](lifecycle-spring-boot.md)

Reference: [Workflows & Steps](https://docs.dbos.dev/java/reference/workflows-steps)
