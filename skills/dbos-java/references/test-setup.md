---
title: Test Workflows with Mockito and Testcontainers
impact: LOW-MEDIUM
impactDescription: Fast unit tests for workflow logic plus real integration tests for durability
tags: testing, junit, mockito, testcontainers, integration
---

## Test Workflows with Mockito and Testcontainers

`DBOS` is an ordinary injected object, not a global, so workflow logic can be unit tested with a mock and no
database. Test durable behavior — recovery, queues, exactly-once steps — with a real PostgreSQL instance.

**Incorrect (tests sharing one live DBOS instance):**

```java
// State leaks between tests, and a failure leaves DBOS running
static DBOS dbos = new DBOS(config);

@Test void testOne() { dbos.launch(); proxy.workflow("a"); }
@Test void testTwo() { proxy.workflow("b"); }
```

**Correct (unit test with a mocked DBOS):**

```java
import static org.mockito.Mockito.*;

class CheckoutWorkflowTest {

  // Mockito needs help with the generic step functional interfaces
  private static ThrowingRunnable<RuntimeException> anyRunnable() {
    return ArgumentMatchers.any();
  }

  private static <T> ThrowingSupplier<T, RuntimeException> anySupplier() {
    return ArgumentMatchers.any();
  }

  private DBOS mockDBOS;
  private OrderService service;

  @BeforeEach
  void setUp() {
    mockDBOS = mock(DBOS.class);
    service = new OrderService(mockDBOS, mock(OrderRepository.class));
    service.setSelf(mock(OrderService.class)); // mocked self-proxy
  }

  @Test
  void checkoutWorkflow_paymentSuccessful_marksOrderPaid() throws Exception {
    when(mockDBOS.runStep(anySupplier(), eq("createOrder"))).thenReturn(42);
    when(mockDBOS.recv(eq(PAYMENT_STATUS), any())).thenReturn(Optional.of("paid"));

    service.checkoutWorkflow();

    InOrder inOrder = Mockito.inOrder(mockDBOS);
    inOrder.verify(mockDBOS).runStep(anyRunnable(), eq("subtractInventory"));
    inOrder.verify(mockDBOS).runStep(anySupplier(), eq("createOrder"));
    inOrder.verify(mockDBOS).runStep(anyRunnable(), eq("markOrderPaid"));
  }
}
```

Because the workflow body only touches its dependencies through `dbos.runStep(() -> ..., name)`, the lambdas are
never executed against a mocked `DBOS` — stub step results to drive each branch. Asserting
`verifyNoInteractions(mockRepo)` in an `@AfterEach` catches workflow code that calls a dependency outside a step.

**Correct (integration test against real PostgreSQL):**

```java
@Testcontainers
class MyWorkflowIntegrationTest {

  @Container
  static final PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:latest");

  @Test
  void myWorkflow_completesSuccessfully() throws Exception {
    DBOSConfig config = DBOSConfig.defaults("test-app")
        .withAppVersion("0.1.0")
        .withDatabaseUrl(postgres.getJdbcUrl())
        .withDbUser(postgres.getUsername())
        .withDbPassword(postgres.getPassword());

    // DBOS is AutoCloseable: shutdown always runs
    try (var dbos = new DBOS(config)) {
      var impl = new MyWorkflowsImpl(dbos);
      var proxy = dbos.registerProxy(MyWorkflows.class, impl);
      impl.setSelf(proxy);
      dbos.launch();

      assertEquals("expected", proxy.myWorkflow("input"));
    }
  }
}
```

Guidelines:

- Create one `DBOS` instance per test (or per class) inside try-with-resources so `shutdown()` always runs
- Use a fresh database or unique application name per test run to avoid interference from leftover workflows
- With Spring Boot, construct the `@Service` directly with mocks rather than loading the context with
  `@SpringBootTest` — it is much faster
- Exercise recovery by cancelling a workflow mid-run and calling `dbos.resumeWorkflow(workflowId)`

Reference: [Testing Workflows](https://docs.dbos.dev/java/tutorials/testing)
