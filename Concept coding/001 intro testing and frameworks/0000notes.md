Here's a clear breakdown of all three testing types, followed by a visual summary:

![alt text](image-1.png)

Mock here means when we call `numUtils.multiply(2,4)` return `8`.

**Unit testing** is the most granular level — you test a single class or method in complete isolation, mocking all external dependencies. The goal is to verify that one unit of logic works correctly on its own.

![alt text](image-2.png)

**Integration testing** verifies that multiple components work correctly *together* — for example, your service layer talking to an actual database, or your controller wiring up correctly. Dependencies are real (or partially real), not mocked.

![alt text](image-3.png)

**Functional testing** (also called end-to-end testing) tests the full application from the outside — HTTP requests in, HTTP responses out — exactly as a real user or client would interact with it. Everything is wired up and running.

---
![alt text](image.png)

### Unit Test — Spring Boot Example

Tests a single service method with dependencies mocked:

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private OrderRepository orderRepository;  // Mocked — no real DB

    @InjectMocks
    private OrderService orderService;

    @Test
    void shouldCalculateTotalPrice() {
        Order order = new Order(List.of(
            new Item("Book", 200.0),
            new Item("Pen",   50.0)
        ));
        when(orderRepository.findById(1L)).thenReturn(Optional.of(order));

        double total = orderService.calculateTotal(1L);

        assertThat(total).isEqualTo(250.0);
        verify(orderRepository).findById(1L); // verify interaction
    }
}
```

Key traits: uses `@ExtendWith(MockitoExtension.class)`, no Spring context loaded, blazing fast.

---

### Integration Test — Spring Boot Example

Tests the service + real repository + an in-memory H2 database together:

```java
@SpringBootTest
@Transactional
class OrderServiceIntegrationTest {

    @Autowired
    private OrderService orderService;   // Real bean

    @Autowired
    private OrderRepository orderRepository; // Real repo, real H2 DB

    @Test
    void shouldSaveAndRetrieveOrder() {
        Order order = new Order(List.of(new Item("Laptop", 80000.0)));
        orderRepository.save(order);

        double total = orderService.calculateTotal(order.getId());

        assertThat(total).isEqualTo(80000.0);
    }
}
```

Key traits: `@SpringBootTest` loads the full context, real DB interactions (H2 in-memory), `@Transactional` rolls back after each test.

#### Why putting `transactional`??

Great question! This is something that confuses a lot of developers. Let me explain it clearly.

---

### First — what happens WITHOUT `@Transactional`?

Every test that saves data to the DB **permanently writes it** and it stays there for the next test.

```java
@SpringBootTest
class UserServiceIntegrationTest {

    @Autowired
    UserRepository userRepository;

    @Test
    void testOne() {
        userRepository.save(new User("Alice"));  // saved to DB ✅
        // test ends — Alice is STILL in DB ❗
    }

    @Test
    void testTwo() {
        userRepository.save(new User("Alice"));  // Alice saved AGAIN!

        long count = userRepository.count();
        assertThat(count).isEqualTo(1);  // ❌ FAILS — count is 2!
                                          // Alice exists twice!
    }
}
```

Tests are **polluting each other** — data from one test bleeds into the next.

---

### WITH `@Transactional` — data is rolled back after each test

```java
@SpringBootTest
@Transactional  // ← magic happens here
class UserServiceIntegrationTest {

    @Autowired
    UserRepository userRepository;

    @Test
    void testOne() {
        userRepository.save(new User("Alice"));  // saved in transaction
        // test ends → transaction ROLLED BACK → Alice deleted from DB 🔄
    }

    @Test
    void testTwo() {
        userRepository.save(new User("Alice"));  // fresh start — DB is clean!

        long count = userRepository.count();
        assertThat(count).isEqualTo(1);  // ✅ PASSES — only one Alice
    }
}
```

After each test method, Spring **automatically rolls back** the transaction — as if the test never happened.

---

### What exactly is a rollback?

Think of it like this — a transaction is a **temporary scratchpad**:

```
Test starts
    │
    ▼
Transaction BEGINS  ──────────────────────────────┐
    │                                             │
    ├── save(user)      → written to scratchpad   │
    ├── save(order)     → written to scratchpad   │  All inside
    ├── update(product) → written to scratchpad   │  one transaction
    │                                             │
Test ends                                         │
    │                                             │
    ▼                                             │
Transaction ROLLED BACK ──────────────────────────┘
    │
    ▼
DB is exactly as it was before the test  ✅
```

Nothing is permanently committed — it's all thrown away.

---

### Real example — why it matters

```java
@SpringBootTest
@Transactional
class OrderServiceIntegrationTest {

    @Autowired
    OrderService orderService;

    @Autowired
    OrderRepository orderRepository;

    @Autowired
    UserRepository userRepository;

    @Test
    void shouldCreateOrderForUser() {
        // Arrange — save test data
        User user = userRepository.save(new User("Bob"));
        
        // Act
        orderService.placeOrder(user.getId(), new Item("Laptop", 80000.0));

        // Assert
        List<Order> orders = orderRepository.findByUserId(user.getId());
        assertThat(orders).hasSize(1)
                          .extracting(Order::getItemName)
                          .containsExactly("Laptop");

        // After this test — EVERYTHING rolled back!
        // Bob deleted, order deleted — DB is clean for next test ✅
    }

    @Test
    void shouldFailWhenUserNotFound() {
        // This test starts with a completely clean DB
        // Bob from previous test is GONE — rolled back ✅

        assertThatThrownBy(() -> orderService.placeOrder(999L, new Item("Phone", 50000.0)))
            .isInstanceOf(UserNotFoundException.class);
    }
}
```

---

### 3 key benefits of `@Transactional` in tests

#### 1. Test isolation — tests don't affect each other
```java
// Each test gets a fresh, clean database state
// No matter what order tests run in — results are the same ✅
```

#### 2. No cleanup code needed
```java
// ❌ WITHOUT @Transactional — you write messy cleanup
@AfterEach
void cleanup() {
    orderRepository.deleteAll();
    userRepository.deleteAll();
    productRepository.deleteAll();
    // forget one → next test breaks!
}

// ✅ WITH @Transactional — no cleanup needed at all!
// Spring handles it automatically after every test
```

#### 3. Tests are repeatable
```java
// Run tests once  → all pass ✅
// Run tests again → all pass ✅
// Run on CI/CD   → all pass ✅
// Always same result — because DB is always clean before each test
```

---

### IMPORTANT — `@Transactional` does NOT work for all cases

#### When your service method itself has `@Transactional`

```java
// Production service
@Service
public class PaymentService {

    @Transactional  // ← service has its own transaction
    public void processPayment(Long orderId) {
        // DB operations here
        orderRepository.updateStatus(orderId, "PAID");
        ledgerRepository.record(orderId);
    }
}
```

```java
// Test
@SpringBootTest
@Transactional
class PaymentServiceIntegrationTest {

    @Test
    void shouldProcessPayment() {
        paymentService.processPayment(1L);
        // ⚠️ The service's @Transactional joins the TEST's transaction
        // So rollback still works here — this is fine ✅
    }
}
```

#### When you use `REQUIRES_NEW` propagation — rollback WON'T work

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void auditLog(String action) {
    // This creates a SEPARATE transaction — it COMMITS independently!
    // Test's @Transactional rollback WON'T undo this!
    auditRepository.save(new AuditLog(action));
}
```

```java
@SpringBootTest
@Transactional
class AuditIntegrationTest {

    @Test
    void testAudit() {
        service.auditLog("USER_LOGIN");
        // ⚠️ AuditLog is COMMITTED to DB — not rolled back!
        // Because REQUIRES_NEW created a separate transaction
        // You need manual cleanup here ❗
    }
}
```

---

### When to use `@Rollback(false)` — keep data after test

Sometimes you actually WANT the data to stay — for debugging:

```java
@SpringBootTest
@Transactional
class OrderServiceIntegrationTest {

    @Test
    @Rollback(false)  // ← data stays in DB after this test
    void shouldCreateOrderAndKeepForDebugging() {
        orderService.placeOrder(userId, item);
        // Data committed — you can inspect DB manually after test runs
        // ⚠️ Remember to clean up manually — this will affect other tests!
    }
}
```

---

### `@Transactional` vs `@Sql` for cleanup

Two approaches to handle test data:

```java
// Approach 1: @Transactional — automatic rollback (most common)
@SpringBootTest
@Transactional
class Test1 { ... }


// Approach 2: @Sql — run SQL scripts to reset DB state
@SpringBootTest
@Sql(scripts = "/reset-db.sql",
     executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)
class Test2 {
    // reset-db.sql runs after each test
    // DELETE FROM orders; DELETE FROM users; etc.
    // More explicit but more work to maintain
}
```

`@Transactional` is simpler — prefer it unless you have `REQUIRES_NEW` scenarios.

---

### Summary

| | Without `@Transactional` | With `@Transactional` |
|---|---|---|
| Data after test | Stays in DB ❌ | Rolled back ✅ |
| Test isolation | Tests affect each other ❌ | Each test is independent ✅ |
| Cleanup code | You write it manually ❌ | Automatic ✅ |
| Repeatable | May fail on re-run ❌ | Always same result ✅ |
| Works with `REQUIRES_NEW` | N/A | ⚠️ Rollback won't work |

---

### Bottom line

> `@Transactional` in integration tests means — **"whatever this test writes to the DB, undo it completely when the test finishes"**
>
> It keeps your database **clean between tests** automatically, so every test starts with a **fresh, predictable state** — no manual cleanup, no test pollution, no order dependency.

#### Why `transactional` roll back in Integration test??

Great question! This is about understanding **how Spring handles `@Transactional` differently** in test context vs production context.

---

### In Production — transaction COMMITS

```java
// Production code
@Service
public class UserService {

    @Transactional
    public void registerUser(User user) {
        userRepository.save(user);
        emailService.sendWelcome(user);
        // method ends → transaction COMMITS → data saved permanently ✅
    }
}
```

Normal flow:
```
Method starts → Transaction begins → DB operations → Method ends → COMMIT ✅
```

---

### In Tests — Spring OVERRIDES the default behavior

When you put `@Transactional` on a **test class or test method**, Spring's test framework **deliberately changes the default from COMMIT to ROLLBACK**.

```java
@SpringBootTest
@Transactional   // ← Spring Test sees this differently than production!
class UserServiceTest {

    @Test
    void shouldSaveUser() {
        userRepository.save(new User("Alice"));
        // method ends → Spring TEST framework → ROLLBACK ❌ (not commit!)
    }
}
```

This is **not** normal `@Transactional` behavior — Spring Test framework **intercepts** it.

---

### How Spring does this internally

Spring Test uses a special class called `TransactionalTestExecutionListener` that wraps every test method:

```
┌─────────────────────────────────────────────┐
│         TransactionalTestExecutionListener  │
│                                             │
│  beforeTest()  → BEGIN transaction          │
│                                             │
│      your @Test method runs here            │
│      → save, update, delete operations      │
│                                             │
│  afterTest()   → ROLLBACK (not commit!)     │
└─────────────────────────────────────────────┘
```

In code terms, Spring is essentially doing this behind the scenes:

```java
// What Spring Test framework does internally for EVERY test
void runTest() {
    entityManager.getTransaction().begin();   // 1. begin transaction

    try {
        yourTestMethod();                      // 2. run your test
    } finally {
        entityManager.getTransaction()
                     .rollback();             // 3. ALWAYS rollback — never commit!
    }
}
```

You never write this — Spring does it **automatically** for every single test method.

---

### Step by step — what happens during a test

```java
@SpringBootTest
@Transactional
class OrderServiceIntegrationTest {

    @Autowired OrderRepository orderRepository;
    @Autowired UserRepository  userRepository;

    @Test
    void shouldPlaceOrder() {
        // STEP 1: Spring begins transaction automatically
        //         (before your first line runs)

        User user = userRepository.save(new User("Alice"));
        // STEP 2: Alice written to DB scratchpad (not committed yet)

        orderRepository.save(new Order(user, "Laptop", 80000.0));
        // STEP 3: Order written to DB scratchpad (not committed yet)

        assertThat(orderRepository.count()).isEqualTo(1); // ✅ passes
        // STEP 4: Can read data within same transaction — it's visible

        // STEP 5: Test method ends
        // STEP 6: Spring automatically calls ROLLBACK
        //         Alice → gone, Order → gone, DB back to original state
    }
}
```

```
Timeline:
─────────────────────────────────────────────────────────────
BEGIN TXN → save(Alice) → save(Order) → assert → END → ROLLBACK
              ↑               ↑                           ↑
          in scratchpad   in scratchpad              everything
                                                       erased
─────────────────────────────────────────────────────────────
```

---

### Why can you still READ data within the same test?

Because all operations happen **inside the same transaction** — reads and writes share the same scratchpad:

```java
@Test
void shouldReadWithinSameTransaction() {
    // Write
    userRepository.save(new User("Bob"));

    // Read — works fine! Same transaction sees its own uncommitted writes
    User found = userRepository.findByName("Bob");
    assertThat(found).isNotNull();  // ✅ passes

    // But if another thread / another transaction looks at DB right now
    // they would NOT see Bob — he's not committed yet!
}
// After test → ROLLBACK → Bob is gone from DB forever
```

---

### What if you have MULTIPLE test methods?

Each test method gets its **own fresh transaction** — completely independent:

```java
@SpringBootTest
@Transactional
class UserServiceIntegrationTest {

    @Test
    void test1() {
        // Transaction A begins
        userRepository.save(new User("Alice"));
        assertThat(userRepository.count()).isEqualTo(1); // ✅
        // Transaction A ROLLED BACK → Alice gone
    }

    @Test
    void test2() {
        // Transaction B begins — fresh start, Alice does NOT exist
        userRepository.save(new User("Bob"));
        assertThat(userRepository.count()).isEqualTo(1); // ✅ not 2!
        // Transaction B ROLLED BACK → Bob gone
    }

    @Test
    void test3() {
        // Transaction C begins — completely clean DB
        assertThat(userRepository.count()).isEqualTo(0); // ✅ empty!
        // Transaction C ROLLED BACK
    }
}
```

```
DB state over time:
─────────────────────────────────────────────────────────
 empty │ test1 runs │ rollback │ test2 runs │ rollback │ test3 runs │ rollback
  DB   │ Alice in   │ Alice    │ Bob in     │ Bob      │ nothing    │
       │ scratchpad │ erased   │ scratchpad │ erased   │ written    │
─────────────────────────────────────────────────────────
```

---

### Production vs Test — side by side

```java
// ── PRODUCTION ──
@Service
public class UserService {
    @Transactional
    public void createUser(User user) {
        userRepository.save(user);
    }   // ← method ends → COMMITS → data permanently saved ✅
}


// ── TEST ──
@SpringBootTest
@Transactional          // same annotation — DIFFERENT behavior!
class UserServiceTest {
    @Test
    void test() {
        userRepository.save(new User("Alice"));
    }   // ← test ends → ROLLS BACK → data erased 🔄
}
```

| | Production `@Transactional` | Test `@Transactional` |
|---|---|---|
| Who controls it | Spring AOP proxy | Spring Test framework |
| Default behavior | COMMIT | ROLLBACK |
| Data after method | Permanently saved | Completely erased |
| Purpose | Business operation | Test isolation |

---

### The class responsible — `TransactionalTestExecutionListener`

Spring Test ships with this built-in listener that handles everything:

```java
// Simplified version of what Spring does internally
public class TransactionalTestExecutionListener {

    public void beforeTestMethod(TestContext context) {
        // Start a new transaction before every @Test method
        transactionManager.beginTransaction();
    }

    public void afterTestMethod(TestContext context) {
        // Check if @Rollback(false) is present
        boolean shouldRollback = !hasRollbackFalseAnnotation(context);

        if (shouldRollback) {
            transactionManager.rollback();   // default — always rollback
        } else {
            transactionManager.commit();     // only if @Rollback(false)
        }
    }
}
```

This listener is **automatically registered** when you use `@SpringBootTest` — you don't configure anything.

---

### Override rollback with `@Rollback(false)`

If for some reason you want the data to persist after a test:

```java
@SpringBootTest
@Transactional
class OrderIntegrationTest {

    @Test
    @Rollback(false)         // ← override: commit instead of rollback
    void shouldPersistOrder() {
        orderRepository.save(new Order("Laptop", 80000.0));
        // test ends → COMMITS → data stays in DB permanently
        // ⚠️ Now you must clean up manually — affects other tests!
    }
}
```

---

### Summary — why rollback happens

```
Normal @Transactional (production)
    → controlled by Spring AOP
    → default = COMMIT

@Transactional on test (test context)
    → controlled by TransactionalTestExecutionListener
    → default = ROLLBACK   ← Spring deliberately changed this!
    → reason = keep tests isolated and DB clean
```

---

### Bottom line

> Spring Test framework **deliberately overrides** the default commit behavior.
>
> When `@Transactional` is on a test, Spring wraps every test method in a transaction and **always rolls it back at the end** — using `TransactionalTestExecutionListener` behind the scenes.
>
> This is **not magic** — it's a conscious design decision by Spring to make integration tests **isolated, repeatable, and side-effect free** without any manual cleanup code.

---

### Functional Test — Spring Boot Example

Tests the entire HTTP layer end-to-end using `MockMvc` or `TestRestTemplate`:

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class OrderControllerFunctionalTest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void shouldReturnOrderTotal_viaHTTP() {
        // First create an order
        OrderRequest req = new OrderRequest(List.of(
            new ItemDto("Headphones", 3000.0)
        ));
        ResponseEntity<OrderResponse> created =
            restTemplate.postForEntity("/api/orders", req, OrderResponse.class);

        assertThat(created.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        Long orderId = created.getBody().getId();

        // Now fetch the total
        ResponseEntity<Double> totalResp =
            restTemplate.getForEntity("/api/orders/" + orderId + "/total", Double.class);

        assertThat(totalResp.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(totalResp.getBody()).isEqualTo(3000.0);
    }
}
```

Key traits: real HTTP calls, full Spring context + embedded server, tests Controller → Service → Repository → DB as one flow.

---

Now here's a visual comparison:---

### Quick comparison table

| | Unit | Integration | Functional |
|---|---|---|---|
| **What's tested** | One class/method | Multiple layers | Full HTTP flow |
| **Dependencies** | All mocked | Real (H2, beans) | All real |
| **Spring context** | Not loaded | Loaded | Loaded + server |
| **Speed** | Very fast (ms) | Moderate (sec) | Slow (sec+) |
| **Annotation** | `@ExtendWith(MockitoExtension.class)` | `@SpringBootTest` | `@SpringBootTest(RANDOM_PORT)` |
| **Best for** | Business logic | DB queries, wiring | API contracts, e2e flows |

### The golden rule

Follow the **testing pyramid** — write many unit tests (cheap, fast, isolated), fewer integration tests, and only a handful of functional tests. Functional tests are the most realistic but also the slowest and most brittle, so use them to validate critical user flows, not every edge case.

# Mock

## What is a Mock?

A **mock** is a fake object that simulates the behavior of a real dependency — so you can test your code in isolation without needing the actual database, HTTP service, email sender, etc.

Think of it like a **stunt double** in a movie. The real actor (your actual dependency) isn't on set — a stand-in performs the scripted actions instead.

---

### Why do we need mocks?

When testing `OrderService`, you don't want it to actually hit a real database because:
- Tests become slow
- Tests can fail due to DB being down (not your code's fault)
- You can't control what data is in the DB
- Side effects (inserting, deleting) pollute other tests

So you **mock** the repository and tell it exactly what to return.

---

### Mockito — the mocking library in Spring Boot

```java
@ExtendWith(MockitoExtension.class)
class PaymentServiceTest {

    @Mock
    private PaymentGateway paymentGateway;  // Fake object — no real API call

    @InjectMocks
    private PaymentService paymentService;  // Real class being tested

    @Test
    void shouldProcessPaymentSuccessfully() {

        // 1. STUBBING — tell the mock what to return when called
        when(paymentGateway.charge(1000.0)).thenReturn("TXN_SUCCESS_123");

        // 2. Call your real service (which internally calls the mock)
        String result = paymentService.processPayment(1000.0);

        // 3. ASSERT the outcome
        assertThat(result).isEqualTo("Payment done: TXN_SUCCESS_123");

        // 4. VERIFY — confirm the mock was actually called
        verify(paymentGateway).charge(1000.0);
    }

    @Test
    void shouldHandlePaymentFailure() {
        // Make the mock throw an exception
        when(paymentGateway.charge(1000.0))
            .thenThrow(new PaymentException("Card declined"));

        assertThatThrownBy(() -> paymentService.processPayment(1000.0))
            .isInstanceOf(PaymentException.class)
            .hasMessage("Card declined");
    }
}
```

---

### The 3 things you do with a mock

| Step | What it means | Mockito syntax |
|---|---|---|
| **Stub** | Define what the mock returns | `when(...).thenReturn(...)` |
| **Act** | Call your real code | `service.doSomething()` |
| **Verify** | Confirm mock was called correctly | `verify(mock).method(args)` |

---

### `@Mock` vs `@Spy` vs `@MockBean`

```java
@Mock
PaymentGateway gateway;
// Completely fake — all methods return null/0/false by default
// You must stub every method you want to use


@Spy
PaymentGateway gateway = new RealPaymentGateway();
// Uses the REAL object, but lets you override specific methods
// Great when you want mostly real behavior + one stubbed method


@MockBean  // Spring Boot specific
PaymentGateway gateway;
// Like @Mock but registers the mock INTO the Spring context
// Use this in @SpringBootTest when other beans need to autowire it
```

---

### Real-world analogy

Imagine you're testing a **coffee machine**:

- You don't want to use a real water supply → mock the water pump → `when(pump.getWater()).thenReturn(200ml)`
- You don't want to grind real beans → mock the grinder → `when(grinder.grind()).thenReturn(finePowder)`
- Now you test *just the machine's logic* — did it combine them correctly? Did it heat to the right temp?

The mock gives you **full control** over inputs so you can focus purely on the logic you're testing.


Excellent question! This is where most developers get confused. Knowing **when to mock and when not to** is the real skill.

---

### The golden rule

> **Mock things your code TALKS TO, not things your code IS.**
>
> Mock **external dependencies** — never mock the class you're actually testing.

---

### WHEN TO MOCK ✅

#### 1. External services / APIs (HTTP calls)
You don't want your test calling a real payment gateway or weather API.

```java
// ❌ WITHOUT mock — real HTTP call, test fails if API is down
class OrderServiceTest {
    OrderService service = new OrderService(new RealPaymentGateway());

    @Test
    void shouldProcessPayment() {
        // Makes actual HTTP call to Stripe/Razorpay — SLOW, UNRELIABLE
        service.processPayment(1000.0);
    }
}

// ✅ WITH mock — no real HTTP call, always fast and reliable
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    PaymentGateway paymentGateway;  // fake — no real API call

    @InjectMocks
    OrderService orderService;

    @Test
    void shouldProcessPayment() {
        when(paymentGateway.charge(1000.0)).thenReturn("TXN_001");

        String result = orderService.processPayment(1000.0);

        assertThat(result).isEqualTo("Payment done: TXN_001");
    }
}
```

---

#### 2. Database / Repository layer
Real DB calls make unit tests slow and require setup/teardown.

```java
// ✅ Mock the repository — test only the service logic
@Mock
UserRepository userRepository;

@Test
void shouldReturnActiveUsersOnly() {
    when(userRepository.findAll()).thenReturn(List.of(
        new User("Alice", true),
        new User("Bob",   false)
    ));

    List<User> result = userService.getActiveUsers();

    assertThat(result).hasSize(1)
                      .extracting(User::getName)
                      .containsExactly("Alice");
}
// No DB needed — tests pure filtering logic in the service
```

---

#### 3. Email / SMS / Notification senders
You never want tests to send real emails or SMS.

```java
// ✅ Mock the sender — no real email sent during tests
@Mock
EmailService emailService;

@Test
void shouldSendWelcomeEmailOnRegistration() {
    userService.register("john@example.com");

    // Just verify it was CALLED with right args — no actual email sent
    verify(emailService).sendWelcome("john@example.com");
}
```

---

#### 4. Clock / Date / Time
Time-dependent code is tricky — mock the clock so tests are deterministic.

```java
// ❌ WITHOUT mock — test breaks at midnight, or on specific dates
public boolean isExpired(LocalDate expiryDate) {
    return LocalDate.now().isAfter(expiryDate); // depends on real time!
}

// ✅ WITH mock — you control what "now" means
@Mock
Clock clock;

@Test
void shouldDetectExpiredSubscription() {
    when(clock.now()).thenReturn(LocalDate.of(2026, 6, 1));

    boolean expired = subscriptionService.isExpired(
        LocalDate.of(2026, 1, 1)
    );

    assertThat(expired).isTrue();  // always passes, regardless of real date
}
```

---

#### 5. File system / IO operations
Don't read/write real files in unit tests.

```java
// ✅ Mock the file reader
@Mock
FileReader fileReader;

@Test
void shouldParseCSVCorrectly() {
    when(fileReader.readLines("data.csv"))
        .thenReturn(List.of("Alice,30", "Bob,25"));

    List<User> users = csvParser.parse("data.csv");

    assertThat(users).hasSize(2);
}
```

---

#### 6. Slow or expensive computations
If a dependency does heavy processing, mock it.

```java
// ✅ Mock the AI/ML model — don't run real inference in unit tests
@Mock
MLModel fraudDetector;

@Test
void shouldBlockFlaggedTransaction() {
    when(fraudDetector.isFraud(any())).thenReturn(true);

    boolean blocked = transactionService.process(transaction);

    assertThat(blocked).isFalse();  // transaction was blocked
}
```

---

### WHEN NOT TO MOCK ❌

#### 1. Never mock the class you're testing
This is the most common mistake beginners make.

```java
// ❌ WRONG — mocking the class under test is pointless!
@Mock
OrderService orderService;  // you're testing THIS class — don't mock it!

@Test
void badTest() {
    when(orderService.getTotal()).thenReturn(500.0);  // you're just
    assertThat(orderService.getTotal()).isEqualTo(500.0); // testing Mockito!
}

// ✅ CORRECT — test the real class, mock its dependencies
@Mock
OrderRepository orderRepository;  // mock the DEPENDENCY

@InjectMocks
OrderService orderService;        // real class under test
```

---

#### 2. Don't mock simple value objects / POJOs
DTOs, entities, models — just create real instances.

```java
// ❌ WRONG — mocking a simple POJO is unnecessary and confusing
@Mock
User user;
when(user.getName()).thenReturn("Alice");

// ✅ CORRECT — just create the object!
User user = new User("Alice", 30, "alice@email.com");
```

---

#### 3. Don't mock pure utility / helper classes
Classes with no external dependencies and simple logic don't need mocking.

```java
// ❌ WRONG — why mock a simple calculator?
@Mock
PriceCalculator calculator;
when(calculator.applyTax(100.0)).thenReturn(118.0);

// ✅ CORRECT — just use the real thing
PriceCalculator calculator = new PriceCalculator();
assertThat(calculator.applyTax(100.0)).isEqualTo(118.0);
```

---

#### 4. Don't mock in integration tests
Integration tests exist to test real wiring — mocking defeats the purpose.

```java
// ❌ WRONG — mocking in an integration test defeats its purpose
@SpringBootTest
class OrderIntegrationTest {
    @MockBean                        // why mock in integration test?
    OrderRepository orderRepository;
}

// ✅ CORRECT — use real repo with H2 in-memory DB
@SpringBootTest
@Transactional
class OrderIntegrationTest {
    @Autowired
    OrderRepository orderRepository; // real repo, real H2 DB
}
```

---

#### 5. Don't mock when the real thing is easy and fast

```java
// ❌ WRONG — overkill to mock a simple list or string
@Mock
List<String> names;
when(names.size()).thenReturn(3);

// ✅ CORRECT — just use the real object
List<String> names = List.of("Alice", "Bob", "Carol");
assertThat(names).hasSize(3);
```

---

### Decision chart — should I mock this?

```
Is this the class I'm testing?
        │
        ├── YES → DON'T mock it (use @InjectMocks)
        │
        └── NO → Is it a dependency?
                    │
                    ├── Does it touch DB / API / File / Email / Time?
                    │         └── YES → MOCK IT ✅
                    │
                    ├── Is it slow or non-deterministic?
                    │         └── YES → MOCK IT ✅
                    │
                    ├── Is it a simple POJO / utility with no side effects?
                    │         └── YES → DON'T mock, use real object ❌
                    │
                    └── Is this an integration test?
                                └── YES → DON'T mock, use real beans ❌
```

---

### Summary table

| Scenario | Mock? | Reason |
|---|---|---|
| Database / Repository | ✅ Yes | Slow, needs setup |
| External HTTP API | ✅ Yes | Unreliable, costs money |
| Email / SMS sender | ✅ Yes | Side effects in real world |
| Clock / Date / Time | ✅ Yes | Non-deterministic |
| File system / IO | ✅ Yes | Slow, needs real files |
| The class under test | ❌ No | You're testing it — keep it real |
| Simple POJO / DTO | ❌ No | Easy to create, no side effects |
| Utility / helper class | ❌ No | Pure logic, fast, no dependencies |
| In integration tests | ❌ No | Real wiring is the whole point |
| Java standard library | ❌ No | Always available, always fast |

---

### Bottom line

> Mock when a dependency is **slow**, **unreliable**, has **side effects**, or is **hard to control** in tests.
>
> Don't mock when the real thing is **simple**, **fast**, and has **no external dependencies**.
>
> The goal of mocking is **isolation** — test one thing at a time, predictably and fast.



# Junit vs Mockito 

JUnit and Mockito are often used **together** but they do completely different jobs.

---

### One-line summary

| | JUnit | Mockito |
|---|---|---|
| **Purpose** | Runs your tests & checks results | Creates fake objects (mocks) |
| **Answers** | *"Did the output match expectation?"* | *"Did the code interact with dependencies correctly?"* |
| **Analogy** | The **judge** — gives the verdict | The **actor** — plays a fake role |

---

### JUnit — the test runner & assertion library

JUnit is the **framework that runs your tests**. It provides:
- `@Test` — marks a method as a test
- `@BeforeEach / @AfterEach` — setup/teardown
- `assertThat`, `assertEquals`, `assertThrows` — check outcomes

```java
@Test
void shouldAddTwoNumbers() {
    Calculator calc = new Calculator();

    int result = calc.add(2, 3);

    // JUnit assertion — did we get what we expected?
    assertEquals(5, result);           // JUnit 5
    assertThat(result).isEqualTo(5);   // AssertJ (works with JUnit)
}
```

JUnit doesn't care *how* the result was produced — it just checks **what came out**.

---

### Mockito — the mocking library

Mockito **creates fake versions** of dependencies so your class under test doesn't need real ones.

```java
@Test
void shouldSendWelcomeEmail() {
    // Mockito creates a fake EmailService
    EmailService emailService = mock(EmailService.class);

    UserService userService = new UserService(emailService);
    userService.registerUser("john@example.com");

    // Mockito verify — was the fake called correctly?
    verify(emailService).sendWelcome("john@example.com");
}
```

Mockito doesn't run tests or make assertions about return values — it controls and **verifies interactions** with dependencies.

---

### Used together — this is how 99% of real tests look

```java
@ExtendWith(MockitoExtension.class)   // ← Mockito plugs INTO JUnit 5
class OrderServiceTest {

    @Mock
    OrderRepository orderRepository;  // Mockito: fake dependency

    @InjectMocks
    OrderService orderService;        // Real class being tested

    @Test                             // JUnit: this is a test
    @DisplayName("Should return total price of order")
    void shouldReturnTotalPrice() {

        // Mockito: stub the fake
        Order order = new Order(List.of(new Item("Phone", 50000.0)));
        when(orderRepository.findById(1L)).thenReturn(Optional.of(order));

        // Call real code
        double total = orderService.getTotal(1L);

        // JUnit: assert the result
        assertThat(total).isEqualTo(50000.0);

        // Mockito: verify the interaction
        verify(orderRepository, times(1)).findById(1L);
    }
}
```

Notice: `@ExtendWith(MockitoExtension.class)` is how Mockito **plugs into** JUnit 5's extension system.

---

### What each line belongs to

```java
@Test                              // ← JUnit
void myTest() {
    when(repo.find()).thenReturn(); // ← Mockito (stubbing)

    service.doWork();              // ← your real code

    assertThat(result).isEqualTo();// ← JUnit / AssertJ
    verify(repo).find();           // ← Mockito (verification)
}
```

---

### Key annotations — who owns what

| Annotation | Library | Purpose |
|---|---|---|
| `@Test` | JUnit 5 | Marks method as a test |
| `@BeforeEach` | JUnit 5 | Runs before every test |
| `@AfterEach` | JUnit 5 | Runs after every test |
| `@DisplayName` | JUnit 5 | Human-readable test name |
| `@ParameterizedTest` | JUnit 5 | Run test with multiple inputs |
| `@Mock` | Mockito | Create a mock object |
| `@Spy` | Mockito | Partial mock of real object |
| `@InjectMocks` | Mockito | Inject mocks into real class |
| `@Captor` | Mockito | Capture arguments passed to mock |
| `@ExtendWith(MockitoExtension.class)` | Both | Connects Mockito to JUnit 5 |

---

### Can you use them separately?

**JUnit without Mockito** — yes, for pure logic with no dependencies:
```java
@Test
void shouldCalculateDiscount() {
    // No mocks needed — pure math
    PriceCalculator calc = new PriceCalculator();
    assertThat(calc.discount(1000, 10)).isEqualTo(900.0);
}
```

**Mockito without JUnit** — technically yes, but pointless in practice. Mockito just creates fakes — you still need *something* to run the test and assert results.

---

### Bottom line

> **JUnit** = the testing framework (runs tests, checks results)
>
> **Mockito** = the mocking library (fakes dependencies, verifies calls)
>
> They complement each other — JUnit gives you the **ring**, Mockito gives you the **stunt doubles**.

# AssertJ

Great question! AssertJ is something most developers overlook at first but love once they discover it.

---

### What is AssertJ?

AssertJ is a **fluent assertion library** for Java. It gives you a much more readable and powerful way to write assertions in your tests compared to plain JUnit assertions.

It's not a test runner (that's JUnit) — it purely focuses on **how you verify results**.

---

### JUnit assertions vs AssertJ — side by side

```java
// ── Plain JUnit 5 assertions ──
assertEquals(200, response.getStatus());
assertTrue(name.contains("John"));
assertNotNull(user);
assertThrows(Exception.class, () -> service.doWork());

// ── Same thing with AssertJ ──
assertThat(response.getStatus()).isEqualTo(200);
assertThat(name).contains("John");
assertThat(user).isNotNull();
assertThatThrownBy(() -> service.doWork()).isInstanceOf(Exception.class);
```

Both do the same job — but AssertJ reads like **plain English**.

---

### Why is AssertJ better? The real reasons

#### 1. Fluent chaining — multiple checks in one line

```java
// JUnit — needs separate assert for each check
assertEquals("John", user.getName());
assertTrue(user.isActive());
assertNotNull(user.getEmail());

// AssertJ — chain them all together
assertThat(user)
    .extracting("name", "active", "email")
    .containsExactly("John", true, "john@example.com");

// Or chain directly
assertThat(user.getName()).isEqualTo("John");
assertThat(user).isNotNull()
                .matches(u -> u.isActive())
                .matches(u -> u.getEmail() != null);
```

---

#### 2. Way better error messages

```java
// JUnit failure message — cryptic
// Expected: <500.0> but was: <450.0>

// AssertJ failure message — tells you EXACTLY what went wrong
assertThat(price).isEqualTo(500.0);
// [Price check] expected: 500.0 but was: 450.0
//               ^^^ you can even add a custom label with .as()

assertThat(price).as("Final price after discount").isEqualTo(500.0);
// [Final price after discount] expected: 500.0 but was: 450.0
```

---

#### 3. Powerful collection assertions

```java
List<String> fruits = List.of("apple", "banana", "mango");

// JUnit — awkward
assertTrue(fruits.contains("apple"));
assertEquals(3, fruits.size());

// AssertJ — expressive and readable
assertThat(fruits)
    .hasSize(3)
    .contains("apple", "mango")
    .doesNotContain("grape")
    .startsWith("apple")
    .endsWith("mango")
    .allMatch(f -> f.length() > 3);
```

---

#### 4. Object field assertions

```java
User user = new User("Alice", 30, "alice@email.com");

// Check specific fields without equals()
assertThat(user)
    .hasFieldOrPropertyWithValue("name", "Alice")
    .hasFieldOrPropertyWithValue("age", 30);

// Extract and assert on a field
assertThat(user)
    .extracting(User::getName)
    .isEqualTo("Alice");
```

---

#### 5. Exception assertions

```java
// JUnit way
assertThrows(UserNotFoundException.class, () -> {
    userService.findById(999L);
});

// AssertJ way — also checks the message!
assertThatThrownBy(() -> userService.findById(999L))
    .isInstanceOf(UserNotFoundException.class)
    .hasMessage("User not found with id: 999")
    .hasMessageContaining("999");

// Alternative style
assertThatExceptionOfType(UserNotFoundException.class)
    .isThrownBy(() -> userService.findById(999L))
    .withMessage("User not found with id: 999");
```

---

#### 6. String assertions

```java
String email = "john.doe@example.com";

assertThat(email)
    .isNotBlank()
    .startsWith("john")
    .endsWith(".com")
    .contains("@")
    .hasSize(20)
    .matches("[a-z.]+@[a-z]+\\.[a-z]+");
```

---

#### 7. Number assertions

```java
double price = 149.99;

assertThat(price)
    .isPositive()
    .isGreaterThan(100.0)
    .isLessThan(200.0)
    .isBetween(100.0, 200.0)
    .isCloseTo(150.0, within(1.0));  // great for floating point!
```

`isCloseTo` is a lifesaver for floating point comparisons where exact equality fails.

---

### Full real-world test using AssertJ

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository userRepository;

    @InjectMocks
    private UserService userService;

    @Test
    void shouldReturnActiveUsersOnly() {
        List<User> users = List.of(
            new User("Alice", true),
            new User("Bob",   false),
            new User("Carol", true)
        );
        when(userRepository.findAll()).thenReturn(users);

        List<User> activeUsers = userService.getActiveUsers();

        assertThat(activeUsers)
            .hasSize(2)
            .extracting(User::getName)          // pull out just names
            .containsExactlyInAnyOrder("Alice", "Carol")
            .doesNotContain("Bob");
    }

    @Test
    void shouldThrowWhenUserNotFound() {
        when(userRepository.findById(99L)).thenReturn(Optional.empty());

        assertThatThrownBy(() -> userService.findById(99L))
            .isInstanceOf(UserNotFoundException.class)
            .hasMessageContaining("99");
    }
}
```

---

### How to add AssertJ to your project

It comes **bundled automatically** with `spring-boot-starter-test` — no extra dependency needed!

```xml
<!-- pom.xml — already included via Spring Boot -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

Just import it in your test class:
```java
import static org.assertj.core.api.Assertions.*;
```

---

### Quick cheat sheet

| What you're checking | AssertJ method |
|---|---|
| Equals | `.isEqualTo(value)` |
| Not null | `.isNotNull()` |
| Null | `.isNull()` |
| Boolean true | `.isTrue()` |
| Contains (string/list) | `.contains(...)` |
| List size | `.hasSize(n)` |
| All elements match | `.allMatch(predicate)` |
| Exception thrown | `assertThatThrownBy(...)` |
| Floating point | `.isCloseTo(val, within(delta))` |
| Custom label | `.as("label").isEqualTo(...)` |

---

### Bottom line

> **JUnit** tells you a test **passed or failed**
>
> **AssertJ** tells you **exactly why it failed**, in human-readable language
>
> Use AssertJ whenever you write assertions — it makes tests easier to read, write, and debug.

Great question! JaCoCo is something every serious Java developer should know.

---

### What is JaCoCo?

**JaCoCo** = **Ja**va **Co**de **Co**verage library.

It's a tool that **measures how much of your production code is actually executed by your tests**. It doesn't check if your tests are *good* — it just tells you which lines, branches, and methods your tests touched.

---

### What is Code Coverage?

Imagine your production code has 100 lines. Your tests only execute 70 of those lines. Your coverage = **70%**.

```
Total lines in production code  = 100
Lines executed by your tests    =  70
─────────────────────────────────────
Coverage                        =  70%
```

Coverage answers: **"Which parts of my code have NO tests at all?"**

---

### Types of coverage JaCoCo measures

#### 1. Line Coverage
Were these lines executed at least once?

```java
public double calculateDiscount(double price, String type) {
    if (type.equals("PREMIUM")) {         // line 1 ✅ tested
        return price * 0.20;              // line 2 ✅ tested
    } else if (type.equals("SEASONAL")) { // line 3 ✅ tested
        return price * 0.10;              // line 4 ❌ NOT tested
    }
    return 0;                             // line 5 ✅ tested
}
// Line coverage = 4/5 = 80%
```

---

#### 2. Branch Coverage
Were ALL branches (if/else, switch) of a condition tested?

```java
if (user.isActive() && user.hasSubscription()) {
    // Branch 1: both true       ✅ tested
    // Branch 2: isActive=false  ❌ NOT tested
    // Branch 3: noSubscription  ❌ NOT tested
    grantAccess();
}
// Branch coverage = 1/3 = 33%  ← much stricter than line coverage!
```

---

#### 3. Method Coverage
Was this method called at least once by any test?

```java
public class OrderService {
    public double getTotal() { ... }    // ✅ called in tests
    public void cancelOrder() { ... }   // ❌ never called in tests
    public void sendInvoice() { ... }   // ✅ called in tests
}
// Method coverage = 2/3 = 66%
```

---

### Setting up JaCoCo in Spring Boot

```xml
<!-- pom.xml -->
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.11</version>
    <executions>

        <!-- Step 1: start recording coverage before tests run -->
        <execution>
            <id>prepare-agent</id>
            <goals><goal>prepare-agent</goal></goals>
        </execution>

        <!-- Step 2: generate HTML report after tests finish -->
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals><goal>report</goal></goals>
        </execution>

        <!-- Step 3: FAIL the build if coverage drops below threshold -->
        <execution>
            <id>check</id>
            <goals><goal>check</goal></goals>
            <configuration>
                <rules>
                    <rule>
                        <element>BUNDLE</element>
                        <limits>
                            <limit>
                                <counter>LINE</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.80</minimum> <!-- 80% minimum -->
                            </limit>
                            <limit>
                                <counter>BRANCH</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.70</minimum> <!-- 70% minimum -->
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>

    </executions>
</plugin>
```

Run it with:
```bash
mvn clean test        # runs tests + generates coverage report
```

Report is generated at:
```
target/site/jacoco/index.html  ← open this in browser
```

---

### What the JaCoCo HTML report looks like

```
Package: com.example.service
─────────────────────────────────────────────────────────
Class               Lines   Branches  Methods   Complexity
─────────────────────────────────────────────────────────
OrderService        85%     72%       90%       78%       ✅
PaymentService      45%     30%       50%       40%       ❌
UserService         92%     88%       100%      90%       ✅
─────────────────────────────────────────────────────────
Total               74%     63%       80%       69%
```

It also **color-codes every line** in your source:
- 🟢 Green = fully covered
- 🟡 Yellow = partially covered (some branches missed)
- 🔴 Red = not covered at all

---

### Real example — coverage in action

```java
// Production code
public class LoanService {

    public String approveLoan(int age, double income, int creditScore) {
        if (age < 18) {                          // branch A
            return "REJECTED: Too young";
        }
        if (income < 30000) {                    // branch B
            return "REJECTED: Low income";
        }
        if (creditScore >= 750) {                // branch C
            return "APPROVED: Excellent credit";
        } else if (creditScore >= 600) {         // branch D
            return "APPROVED: Good credit";
        } else {
            return "REJECTED: Poor credit";      // branch E
        }
    }
}
```

```java
// Test file
@Test
void testApproval() {
    // Only tests branch C
    assertThat(service.approveLoan(25, 50000, 800))
        .isEqualTo("APPROVED: Excellent credit");
}

// JaCoCo report:
// Lines covered:    4/7  = 57%  ❌
// Branches covered: 1/5  = 20%  ❌
// Missing: age<18, income<30000, creditScore>=600, poor credit
```

After adding more tests:
```java
@Test void shouldRejectUnderAge()    { service.approveLoan(16, 50000, 800); }

@Test void shouldRejectLowIncome()   { service.approveLoan(25, 20000, 800); }

@Test void shouldApproveGoodCredit() { service.approveLoan(25, 50000, 680); }

@Test void shouldRejectPoorCredit()  { service.approveLoan(25, 50000, 500); }

// JaCoCo report now:
// Lines covered:    7/7  = 100% ✅
// Branches covered: 5/5  = 100% ✅
```

---

### Excluding classes from coverage

Not everything needs to be covered — DTOs, config classes, generated code:

```java
// Option 1: annotation on the class
@Generated  // JaCoCo ignores classes marked with this
public class UserDTO { ... }

// Option 2: exclude in pom.xml
<configuration>
    <excludes>
        <exclude>com/example/dto/**</exclude>
        <exclude>com/example/config/**</exclude>
        <exclude>com/example/**/model/**</exclude>
    </excludes>
</configuration>
```

---

### Coverage — common misconceptions

| Misconception | Reality |
|---|---|
| 100% coverage = no bugs | ❌ Wrong — tests can execute code without asserting correctly |
| Low coverage = bad tests | ⚠️ Partially — it means untested paths exist |
| Coverage measures test quality | ❌ Wrong — it only measures which code was *touched* |
| You must hit 100% always | ❌ Wrong — 70–80% is a healthy industry standard |

```java
// This gives 100% line coverage but tests NOTHING useful!
@Test
void badTest() {
    service.processOrder(null);  // executes the line — but no assertion!
}
// JaCoCo says ✅ but the test is worthless
```

---

### Industry standard coverage targets

```
< 50%   →  🔴  Danger zone — huge untested areas
50–70%  →  🟡  Acceptable for legacy / early projects  
70–80%  →  🟢  Good — healthy standard for most teams
80–90%  →  🟢  Great — most critical paths covered
> 90%   →  🟢  Excellent — aim here for critical services
100%    →  ⚠️  Overkill — diminishing returns, may slow dev
```

---

### JaCoCo + CI/CD pipeline

```yaml
# GitHub Actions example
- name: Run tests with coverage
  run: mvn clean verify

- name: Upload JaCoCo report
  uses: actions/upload-artifact@v3
  with:
    name: jacoco-report
    path: target/site/jacoco/

# Build FAILS automatically if coverage drops below
# the threshold you set in pom.xml — no manual check needed!
```

---

### Bottom line

> **JaCoCo** watches your tests run and highlights every line of code they **didn't touch**
>
> **Coverage** is a safety net — not a quality guarantee. High coverage means fewer *blind spots*, not perfect tests.
>
> Use JaCoCo to find **untested code paths**, then write meaningful assertions for them — not just to hit a number.