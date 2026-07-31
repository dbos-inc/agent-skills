---
title: Name Class Instances When Registering Multiple Copies
impact: MEDIUM
impactDescription: Recovery routes workflows back to the correct configured instance
tags: workflow, instances, registerProxy, instanceName, configuration
---

## Name Class Instances When Registering Multiple Copies

When the same workflow class is instantiated more than once — one per tenant, region, or downstream service — each
instance must be registered with a unique name. DBOS stores that name with every workflow record so recovery can
resume the workflow on the instance that holds the right configuration.

**Incorrect (multiple unnamed instances of the same class):**

```java
// Both registrations use the default instance name — DBOS cannot tell them
// apart during recovery, and workflows may resume on the wrong configuration
dbos.registerProxy(DataProcessor.class, new DataProcessorImpl("https://service-a.example.com"));
dbos.registerProxy(DataProcessor.class, new DataProcessorImpl("https://service-b.example.com"));
```

**Correct (each instance registered with a unique name):**

```java
DataProcessor processorA = dbos.registerProxy(
    DataProcessor.class,
    new DataProcessorImpl("https://service-a.example.com"),
    "service-a");

DataProcessor processorB = dbos.registerProxy(
    DataProcessor.class,
    new DataProcessorImpl("https://service-b.example.com"),
    "service-b");

dbos.launch();

processorA.process("job-1"); // recovers on the "service-a" instance
processorB.process("job-2"); // recovers on the "service-b" instance
```

Rules and related APIs:

- Register every named instance before `launch()`; recovery fails for an instance name that is not registered
- Instance configuration must be constructor-supplied and stable — workflows must not mutate it
- Target a specific instance from outside the app with
  `new DBOSClient.EnqueueOptions(workflowName, className, queueName).withInstanceName("service-a")`
- Filter by instance with `new ListWorkflowsInput().withInstanceName("service-a")`
- Annotate the implementation with `@WorkflowClassName("data-processor")` to give it a short, stable class name that
  survives package renames and works across languages
- With Spring Boot, the `@Primary` bean of a class uses the default instance name and any additional beans of that
  class are registered under their Spring bean names

Reference: [Workflows on Class Instances](https://docs.dbos.dev/java/tutorials/workflow-classes)
