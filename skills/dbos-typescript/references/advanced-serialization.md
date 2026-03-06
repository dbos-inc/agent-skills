---
title: Use Portable Serialization for Cross-Language Interoperability
impact: LOW
impactDescription: Enables TypeScript workflows to exchange data with Python, Java, and Go applications
tags: serialization, portable, cross-language, interoperability, json
---

## Use Portable Serialization for Cross-Language Interoperability

By default, TypeScript DBOS uses SuperJSON serialization, which only TypeScript can read. Use `"portable"` serialization to write data as JSON that any DBOS SDK (Python, TypeScript, Java, Go) can read and write.

**Incorrect (default SuperJSON — blocks cross-language access):**

```typescript
// A Python or Java client cannot read this workflow's
// inputs, outputs, events, or streams
async function processOrderFn(orderId: string) {
  await DBOS.setEvent("status", { progress: 50 });
  return { result: "done" };
}
const processOrder = DBOS.registerWorkflow(processOrderFn);
```

**Correct (portable JSON — readable by any language):**

```typescript
import { DBOS } from "@dbos-inc/dbos-sdk";

async function processOrderFn(orderId: string) {
  await DBOS.setEvent("status", { progress: 50 });
  return { result: "done" };
}
const processOrder = DBOS.registerWorkflow(processOrderFn, {
  name: "processOrder",
  serializationType: "portable",
});
```

### Supported Portable Types

Portable JSON supports JSON primitives, arrays, and objects. Some TypeScript types are automatically converted:

| TypeScript Type | Portable Representation |
|-----------------|------------------------|
| `Date` | RFC 3339 UTC string |
| `BigInt` | Numeric string |

### Where to Set Serialization

**On workflow registration** — affects inputs, outputs, events, and streams for that workflow:

```typescript
const myWorkflow = DBOS.registerWorkflow(myWorkflowFn, {
  name: "myWorkflow",
  serializationType: "portable",
});
```

Or with a decorator:

```typescript
class Orders {
  @DBOS.workflow({ serializationType: "portable" })
  static async processOrder(orderId: string) {
    await DBOS.setEvent("progress", 50);  // Portable by default
    return { done: true };                // Portable by default
  }
}
```

**On individual operations** — override per-operation when mixing strategies:

```typescript
// Explicitly set portable on send (send is never affected by workflow default)
await DBOS.send(
  "workflow-123",
  { status: "complete" },
  "updates",
  undefined,  // idempotencyKey
  { serializationType: "portable" }
);

// Override on setEvent or writeStream
await DBOS.setEvent("key", value, { serializationType: "portable" });
await DBOS.writeStream("key", value, { serializationType: "portable" });
```

**On enqueue from DBOSClient** — for cross-language workflow submission:

```typescript
import { DBOSClient } from "@dbos-inc/dbos-sdk";

const client = await DBOSClient.create({ systemDatabaseUrl: dbUrl });
const handle = await client.enqueue(
  {
    workflowName: "processOrder",
    queueName: "orders",
    serializationType: "portable",
  },
  "order-123",
);
```

### Serialization Strategy Options

```typescript
// On workflow registration or decorator
serializationType: undefined   // Uses config serializer (SuperJSON by default)
serializationType: "portable"  // Portable JSON for cross-language use
serializationType: "native"    // Explicitly uses native TypeScript serializer
```

### Portable Errors

When a portable workflow fails, the error is serialized in a standard JSON structure all languages can read:

```typescript
import { PortableWorkflowError } from "@dbos-inc/dbos-sdk";

throw new PortableWorkflowError(
  "Order not found",
  "NotFoundError",
  404,
  { orderId: "order-123" },
);
```

Non-portable exceptions raised in a portable workflow are automatically converted to this format on a best-effort basis.

### Key Rules

- `send` is **never** affected by the workflow's serialization strategy — always set `serializationType` explicitly on `send` for cross-language messages
- Step outputs always use the native serializer regardless of workflow strategy (steps are internal)
- `DBOSClient.serializer` must match the app's serializer for **default**-format data, but portable data is always readable

Reference: [Cross-Language Interaction](https://docs.dbos.dev/explanations/portable-workflows)
