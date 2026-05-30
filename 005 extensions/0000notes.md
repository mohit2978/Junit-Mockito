**Extensions** are one of the most powerful features of JUnit 5. Let me explain everything in complete detail.

---

### What is an Extension?

> An Extension is a way to **plug custom behavior into JUnit's test lifecycle** — without modifying your test classes.

Think of it like a **plugin system** for your tests.

```
Without Extension:        With Extension:
────────────────          ──────────────────────────────
@BeforeEach               Extension runs BEFORE @BeforeEach
@Test          →          @Test
@AfterEach                Extension runs AFTER @AfterEach
                          Extension can SKIP test conditionally
                          Extension can LOG every method call
                          Extension can INJECT custom objects
```

---

### How to use an Extension

```java
// Register on a single class
@ExtendWith(MockitoExtension.class)   // you already use this!
class MyTest { ... }

// Register multiple extensions
@ExtendWith({
    MockitoExtension.class,
    MyCustomExtension.class,
    TimingExtension.class
})
class MyTest { ... }

// Register globally — applies to ALL tests
// In: /src/test/resources/junit-platform.properties
junit.extensions.autodetection.enabled = true
```

---

### Extension Points — what you can hook into

JUnit 5 provides many interfaces your extension can implement:

```java
// Each interface = one hook point in the test lifecycle

BeforeAllCallback          // before @BeforeAll
BeforeEachCallback         // before @BeforeEach
BeforeTestExecutionCallback// just before @Test method
TestExecutionExceptionHandler // when test throws exception
AfterTestExecutionCallback // just after @Test method
AfterEachCallback          // after @AfterEach
AfterAllCallback           // after @AfterAll
ExecutionCondition         // should this test run at all?
ParameterResolver          // inject custom params into test
TestInstancePostProcessor  // after test class instantiated
```

---

### Extension Use Case 1 — Add custom code BEFORE/AFTER tests (Timing)

```java
// Custom extension — measures how long each test takes
public class TimingExtension
    implements BeforeTestExecutionCallback,
               AfterTestExecutionCallback {

    private static final String START_TIME = "startTime";

    @Override
    public void beforeTestExecution(ExtensionContext context) {
        // Runs just BEFORE each @Test method
        long startTime = System.currentTimeMillis();

        // Store in JUnit's context store
        context.getStore(ExtensionContext.Namespace.GLOBAL)
               .put(START_TIME, startTime);

        System.out.println("▶ Starting: "
            + context.getDisplayName());
    }

    @Override
    public void afterTestExecution(ExtensionContext context) {
        // Runs just AFTER each @Test method
        long startTime = context.getStore(
            ExtensionContext.Namespace.GLOBAL)
            .get(START_TIME, Long.class);

        long duration = System.currentTimeMillis() - startTime;

        System.out.println("⏱ Finished: "
            + context.getDisplayName()
            + " in " + duration + "ms");

        // Flag slow tests
        if (duration > 1000) {
            System.out.println("⚠️ SLOW TEST DETECTED: "
                + context.getDisplayName());
        }
    }
}
```

```java
// Using the extension
@ExtendWith(TimingExtension.class)
class OrderServiceTest {

    @Test
    void shouldCalculateTotal() {
        // TimingExtension fires BEFORE this
        orderService.calculateTotal(items);
        // TimingExtension fires AFTER this
    }
    // Output:
    // ▶ Starting: shouldCalculateTotal()
    // ⏱ Finished: shouldCalculateTotal() in 45ms
}
```

---

### Extension Use Case 2 — Enable/Disable tests CONDITIONALLY

```java
// Custom extension — skip tests if external service is down
public class ExternalServiceCondition implements ExecutionCondition {

    @Override
    public ConditionEvaluationResult evaluateExecutionCondition(
            ExtensionContext context) {

        // Check if payment gateway is reachable
        boolean isServiceUp = checkPaymentGateway();

        if (isServiceUp) {
            return ConditionEvaluationResult
                .enabled("Payment gateway is UP — running test");
        } else {
            return ConditionEvaluationResult
                .disabled("Payment gateway is DOWN — skipping test");
            // test shows as SKIPPED ⏭️ not FAILED ❌
        }
    }

    private boolean checkPaymentGateway() {
        try {
            // ping the service
            HttpClient.get("https://payment-gateway/health");
            return true;
        } catch (Exception e) {
            return false;
        }
    }
}
```

```java
// Using the conditional extension
@ExtendWith(ExternalServiceCondition.class)
class PaymentIntegrationTest {

    @Test
    void shouldProcessPayment() {
        // Only runs if payment gateway is UP
        // Automatically SKIPPED if gateway is DOWN
        paymentService.charge(1000.0);
    }
}
```

Built-in conditional extensions JUnit provides:

```java
@EnabledOnOs(OS.LINUX)           // only on Linux
@EnabledOnOs(OS.WINDOWS)         // only on Windows
@EnabledOnJre(JRE.JAVA_17)       // only on Java 17
@EnabledIfSystemProperty(
    named = "env",
    matches = "prod")             // only if system property set
@EnabledIfEnvironmentVariable(
    named = "CI",
    matches = "true")             // only in CI environment
@Disabled("Not implemented yet") // always skip
```

---

### Extension Use Case 3 — Intercept method calls for LOGGING

```java
// Custom extension — logs every test method call
public class LoggingExtension
    implements BeforeAllCallback,
               BeforeEachCallback,
               AfterEachCallback,
               AfterAllCallback,
               TestExecutionExceptionHandler {

    @Override
    public void beforeAll(ExtensionContext context) {
        System.out.println("\n════════════════════════════");
        System.out.println("TEST CLASS: "
            + context.getRequiredTestClass().getSimpleName());
        System.out.println("════════════════════════════");
    }

    @Override
    public void beforeEach(ExtensionContext context) {
        System.out.println("\n  ┌─ TEST: "
            + context.getDisplayName());
        System.out.println("  │  Thread: "
            + Thread.currentThread().getName());
    }

    @Override
    public void afterEach(ExtensionContext context) {
        // Check if test passed or failed
        boolean failed = context.getExecutionException().isPresent();

        if (failed) {
            System.out.println("  └─ RESULT: ❌ FAILED");
        } else {
            System.out.println("  └─ RESULT: ✅ PASSED");
        }
    }

    @Override
    public void handleTestExecutionException(
            ExtensionContext context,
            Throwable throwable) throws Throwable {

        // Intercept exception — log it — then rethrow
        System.out.println("  │  EXCEPTION: "
            + throwable.getClass().getSimpleName()
            + " - " + throwable.getMessage());

        throw throwable; // must rethrow or test won't fail!
    }

    @Override
    public void afterAll(ExtensionContext context) {
        System.out.println("\n════════════════════════════\n");
    }
}
```

```java
// Output when tests run:
// ════════════════════════════
// TEST CLASS: OrderServiceTest
// ════════════════════════════
//
//   ┌─ TEST: shouldCalculateTotal()
//   │  Thread: main
//   └─ RESULT: ✅ PASSED
//
//   ┌─ TEST: shouldThrowForInvalidInput()
//   │  Thread: main
//   │  EXCEPTION: IllegalArgumentException - Age cannot be negative
//   └─ RESULT: ❌ FAILED
//
// ════════════════════════════
```

---

### Extension Use Case 4 — Inject custom parameters

```java
// Custom extension — injects a database connection into tests
public class DatabaseExtension
    implements ParameterResolver,
               BeforeAllCallback,
               AfterAllCallback {

    private DatabaseConnection connection;

    @Override
    public void beforeAll(ExtensionContext context) {
        // Create connection once for all tests
        connection = DatabaseConnection.create("jdbc:h2:mem:test");
        connection.runMigrations();
    }

    @Override
    public boolean supportsParameter(
            ParameterContext paramContext,
            ExtensionContext extContext) {
        // Tell JUnit: "I can provide DatabaseConnection params"
        return paramContext.getParameter()
                           .getType() == DatabaseConnection.class;
    }

    @Override
    public Object resolveParameter(
            ParameterContext paramContext,
            ExtensionContext extContext) {
        // Provide the actual value
        return connection;
    }

    @Override
    public void afterAll(ExtensionContext context) {
        connection.close();
    }
}
```

```java
// Using parameter injection extension
@ExtendWith(DatabaseExtension.class)
class DatabaseTest {

    @Test
    void shouldSaveUser(DatabaseConnection db) {
        // ↑ JUnit injects this automatically via extension!
        db.save(new User("Alice", 25));
        assertThat(db.count("users")).isEqualTo(1);
    }

    @Test
    void shouldFindUser(DatabaseConnection db) {
        // same db injected here too
        User found = db.findByName("Alice");
        assertThat(found).isNotNull();
    }
}
```

---

### Real built-in extensions you already use

```java
// MockitoExtension — most common
@ExtendWith(MockitoExtension.class)
// Does:
// → Processes @Mock annotations → creates mock objects
// → Processes @InjectMocks      → injects mocks into class
// → Processes @Captor           → creates argument captors
// → Validates mock usage after each test

// SpringExtension — for Spring Boot tests
@ExtendWith(SpringExtension.class)
// Does:
// → Loads Spring ApplicationContext
// → Processes @Autowired annotations
// → Manages @Transactional rollback
// (included automatically in @SpringBootTest)
```

---

### Create your OWN annotation with Extension

```java
// Step 1: Create custom annotation
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@ExtendWith(TimingExtension.class)  // links annotation to extension
public @interface Timed {
    // custom annotation that activates TimingExtension
}

// Step 2: Use your annotation — cleaner than @ExtendWith!
@Timed                              // ← your annotation
class OrderServiceTest {

    @Test
    void shouldCalculateTotal() {
        // TimingExtension automatically active ✅
    }
}

// Much cleaner than:
@ExtendWith(TimingExtension.class)  // verbose
class OrderServiceTest { ... }
```

---

### Extension lifecycle — full picture

```
@BeforeAll
    │
    ├── BeforeAllCallback.beforeAll()       ← Extension hook
    │
@BeforeEach
    │
    ├── BeforeEachCallback.beforeEach()     ← Extension hook
    │
    ├── BeforeTestExecutionCallback         ← Extension hook
    │
@Test ← your test method runs here
    │
    ├── (if exception) ExceptionHandler     ← Extension hook
    │
    ├── AfterTestExecutionCallback          ← Extension hook
    │
@AfterEach
    │
    ├── AfterEachCallback.afterEach()       ← Extension hook
    │
@AfterAll
    │
    └── AfterAllCallback.afterAll()         ← Extension hook
```

---

### Summary table

| Extension Interface | When it fires | Use for |
|---|---|---|
| `BeforeAllCallback` | Before `@BeforeAll` | Global setup, DB start |
| `BeforeEachCallback` | Before `@BeforeEach` | Logging, setup |
| `BeforeTestExecutionCallback` | Just before `@Test` | Timing start |
| `ExecutionCondition` | Before test runs | Skip/enable conditionally |
| `ParameterResolver` | Test method called | Inject custom params |
| `TestExecutionExceptionHandler` | When test throws | Log/handle exceptions |
| `AfterTestExecutionCallback` | Just after `@Test` | Timing end |
| `AfterEachCallback` | After `@AfterEach` | Cleanup, logging |
| `AfterAllCallback` | After `@AfterAll` | DB stop, global cleanup |
| `TestInstancePostProcessor` | After class created | Inject into test fields |

---

### Bottom line

> JUnit 5 **Extension** is a plugin system that lets you hook into every phase of the test lifecycle
>
> **3 main uses** from the diagram — conditionally enable/disable tests, add code before/after tests, intercept method calls for logging
>
> You implement **one or more Extension interfaces**, register with `@ExtendWith`, and JUnit calls your code automatically at the right moment
>
> The most famous extension you already use daily is **`MockitoExtension`** — it processes `@Mock` and `@InjectMocks` by implementing `BeforeEachCallback` and `ParameterResolver` internally