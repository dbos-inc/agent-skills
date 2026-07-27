---
title: Use Kotlin Extensions for Steps and Workflows
impact: LOW
impactDescription: Idiomatic trailing-lambda syntax without extra dependencies
tags: advanced, kotlin, extensions, lambda, jvm
---

## Use Kotlin Extensions for Steps and Workflows

Every DBOS Java API works from Kotlin unchanged. The `transact` artifact also ships Kotlin extension functions that
put the lambda last so trailing-lambda syntax works; they are `@JvmSynthetic` and invisible to Java callers.

**Incorrect (wrapping lambdas as Java functional interfaces):**

```kotlin
// Argument-order noise: the lambda cannot be a trailing lambda here
val result = dbos.runStep(ThrowingSupplier { fetchFromApi(orderId) }, "fetchOrder")
```

**Correct (extension functions with trailing lambdas):**

```kotlin
interface OrderService {
    fun processOrder(orderId: String): String
}

class OrderServiceImpl(private val dbos: DBOS) : OrderService {

    private lateinit var self: OrderService

    fun setSelf(proxy: OrderService) { self = proxy }

    @Workflow
    override fun processOrder(orderId: String): String {
        val order = dbos.runStep("fetchOrder") {
            fetchFromApi(orderId)
        }
        dbos.runStep(StepOptions("saveOrder").withMaxAttempts(3)) {
            saveToDatabase(order)
        }
        return order
    }
}

val dbos = DBOS(DBOSConfig.defaultsFromEnv("my-app").withAppVersion("0.1.0"))
val impl = OrderServiceImpl(dbos)
val service = dbos.registerProxy(OrderService::class.java, impl)
impl.setSelf(service)
dbos.launch()

val handle = dbos.startWorkflow(StartWorkflowOptions()) {
    service.processOrder("order-123")
}
val result = handle.result
```

Available extensions:

```kotlin
fun <T> DBOS.runStep(name: String, block: () -> T): T
fun <T> DBOS.runStep(options: StepOptions, block: () -> T): T
fun <T> DBOS.startWorkflow(options: StartWorkflowOptions?, block: () -> T): WorkflowHandle<T, Exception>
```

Notes:

- No extra dependency is required — the extensions live in the main `dev.dbos:transact` artifact
- `startWorkflow` always takes an options argument first (pass `StartWorkflowOptions()` or `null`); a zero-argument
  trailing-lambda form is not provided because Kotlin's SAM conversion would resolve to the Java overload
- Registration, lifecycle, queues, and every other API behave exactly as in Java, including the rule that workflows
  and `@Step` methods must be invoked through the registered proxy
- `DBOS` is `AutoCloseable`, so `dbos.use { ... }` shuts it down

Reference: [Using DBOS with Kotlin](https://docs.dbos.dev/java/tutorials/kotlin)
