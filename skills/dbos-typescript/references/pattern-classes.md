---
title: Use DBOS with Class Instances
impact: MEDIUM
impactDescription: Enables configurable workflow instances with recovery support
tags: pattern, class, instance, ConfiguredInstance
---

## Use DBOS with Class Instances

Class instance methods can be workflows and steps. Classes with workflow methods must extend `ConfiguredInstance` to enable recovery.

**Incorrect (instance workflows without ConfiguredInstance):**

```typescript
class MyWorker {
  constructor(private config: any) {}

  @DBOS.workflow()
  async processTask(task: string) {
    // Recovery won't work - DBOS can't find the instance after restart
  }
}
```

**Correct (extending ConfiguredInstance):**

```typescript
import { DBOS, ConfiguredInstance } from "@dbos-inc/dbos-sdk";

class MyWorker extends ConfiguredInstance {
  cfg: WorkerConfig;

  constructor(name: string, config: WorkerConfig) {
    super(name); // Unique name required for recovery
    this.cfg = config;
  }

  override async initialize(): Promise<void> {
    // Optional: validate config at DBOS.launch() time
  }

  @DBOS.workflow()
  async processTask(task: string): Promise<void> {
    // Can use this.cfg safely - instance is recoverable
    const result = await DBOS.runStep(
      () => fetch(this.cfg.apiUrl).then(r => r.text()),
      { name: "callApi" }
    );
  }
}

// Create instances BEFORE DBOS.launch()
const worker1 = new MyWorker("worker-us", { apiUrl: "https://us.api.com" });
const worker2 = new MyWorker("worker-eu", { apiUrl: "https://eu.api.com" });

// Then launch
await DBOS.launch();
```

Key requirements:
- `ConfiguredInstance` constructor requires a unique `name` per class
- All instances must be created **before** `DBOS.launch()`
- The `initialize()` method is called during launch for validation
- Use `DBOS.runStep` inside instance workflows for step operations
- `@DBOS.step()` on an instance method also requires the class to extend `ConfiguredInstance` (`DBOS.runStep` has no such requirement)
- Event receiver decorators such as the Kafka `@consumer` decorator cannot be applied to instance methods, and scheduled workflows and debounced workflows cannot be instance methods

To register an instance method without decorators, register it on the prototype and pass `ctorOrProto` so DBOS can find the instance on dequeue or recovery:

```typescript
class MyWorker extends ConfiguredInstance {
  constructor(name: string, private cfg: WorkerConfig) { super(name); }
  async processTask(task: string): Promise<void> { /* ... */ }
}

MyWorker.prototype.processTask = DBOS.registerWorkflow(MyWorker.prototype.processTask, {
  name: "processTask",
  ctorOrProto: MyWorker.prototype,
});
```

Reference: [Using TypeScript Objects](https://docs.dbos.dev/typescript/tutorials/instantiated-objects)
