### Mockito Verification

> **Video:** [Mockito: Different Ways of Verification](https://www.youtube.com/watch?v=wasBpF2fhwg&list=PL6W8uoQQ2c61_unqN5bY0kYp65Dt_Sm3c&index=19)
>
> **Timestamp:** [00:24](https://www.youtube.com/watch?v=wasBpF2fhwg&t=24s)

### Video Topic Index

| Timestamp | Topic |
|---|---|
| [00:24](https://www.youtube.com/watch?v=wasBpF2fhwg&t=24s) | Introduction to Mockito verification |
| [02:06](https://www.youtube.com/watch?v=wasBpF2fhwg&t=126s) | Verification methods overview |
| [04:40](https://www.youtube.com/watch?v=wasBpF2fhwg&t=280s) | Verifying a method invocation |
| [07:01](https://www.youtube.com/watch?v=wasBpF2fhwg&t=421s) | How calls on a mock are recorded |
| [08:43](https://www.youtube.com/watch?v=wasBpF2fhwg&t=523s) | Exact invocation count with `times(n)` |
| [08:55](https://www.youtube.com/watch?v=wasBpF2fhwg&t=535s) | Verifying that a method was never called |
| [09:24](https://www.youtube.com/watch?v=wasBpF2fhwg&t=564s) | `atLeastOnce()` and `atLeast(n)` |
| [10:21](https://www.youtube.com/watch?v=wasBpF2fhwg&t=621s) | `atMostOnce()` and `atMost(n)` |
| [10:41](https://www.youtube.com/watch?v=wasBpF2fhwg&t=641s) | Meaning of `only()` |
| [10:39](https://www.youtube.com/watch?v=wasBpF2fhwg&t=639s) | Using argument matchers during verification |
| [12:19](https://www.youtube.com/watch?v=wasBpF2fhwg&t=739s) | Matcher examples and custom argument validation |
| [16:38](https://www.youtube.com/watch?v=wasBpF2fhwg&t=998s) | Limitation of validating parameters separately |
| [19:20](https://www.youtube.com/watch?v=wasBpF2fhwg&t=1160s) | Cross-parameter validation with captured arguments |
| [21:32](https://www.youtube.com/watch?v=wasBpF2fhwg&t=1292s) | Verifying method-invocation order |
| [24:06](https://www.youtube.com/watch?v=wasBpF2fhwg&t=1446s) | Order-verification test example |
| [25:57](https://www.youtube.com/watch?v=wasBpF2fhwg&t=1557s) | Using `InOrder` |
| [30:25](https://www.youtube.com/watch?v=wasBpF2fhwg&t=1825s) | `verifyNoInteractions()` |
| [32:34](https://www.youtube.com/watch?v=wasBpF2fhwg&t=1954s) | `verifyNoMoreInteractions()` |
| [34:39](https://www.youtube.com/watch?v=wasBpF2fhwg&t=2079s) | Asynchronous service example |
| [36:27](https://www.youtube.com/watch?v=wasBpF2fhwg&t=2187s) | Why immediate verification can fail for background work |
| [39:00](https://www.youtube.com/watch?v=wasBpF2fhwg&t=2340s) | Difference between `timeout()` and `after()` |
| [40:55](https://www.youtube.com/watch?v=wasBpF2fhwg&t=2455s) | When interaction verification is important |

**JUnit `assertEquals()` vs Mockito `verify()`** — these answer two different questions:

- **JUnit Assertions** check the **output/state**: "did the method *return* the right value?"
- **Mockito `verify()`** checks **behavior/interactions**: "did the method under test *call* this dependency, with what arguments, how many times?"

> Sometimes a method under test doesn't return anything useful (`void`), or the important thing isn't the return value but *whether a dependency was talked to correctly* — e.g. did we actually call `repository.save(user)`? That's what `verify()` is for.

---

### Verification Cheatsheet

> **Timestamp:** [02:06](https://www.youtube.com/watch?v=wasBpF2fhwg&t=126s)

| Category | Method | Meaning |
|---|---|---|
| **Basic** | `verify(mock).method()` | Called exactly once |
| | `verify(mock, times(n)).method()` | Called exactly `n` times |
| | `verify(mock, never()).method()` | Never called |
| **Count** | `verify(mock, atLeastOnce()).method()` | Called 1 or more times |
| | `verify(mock, atLeast(n)).method()` | Called `n` or more times |
| | `verify(mock, atMostOnce()).method()` | Called 0 or 1 times |
| | `verify(mock, atMost(n)).method()` | Called at most `n` times |
| | `verify(mock, only()).method()` | This was the **only** call made to the mock, and it was called once |
| **Argument Matching** | `any()` / `anyInt()` / `anyString()`... | Matches any value of that type |
| | `eq(value)` | Matches an exact value |
| | `isNull()` | Matches a null argument |
| | `argThat(condition)` | Matches an `Object` argument satisfying a custom predicate |
| | `ArgumentCaptor<T>` | Captures the actual argument(s) passed, for later assertions |
| **Order** | `InOrder` | Verifies calls happened in a specific sequence |
| **No Interactions** | `verifyNoInteractions(mock)` | Mock was never touched at all |
| | `verifyNoMoreInteractions(mock)` | No interactions beyond the ones already verified |
| **Async** | `verify(mock, timeout(ms)).method()` | Polls up to `ms`, succeeds as soon as the call is detected |
| | `verify(mock, after(ms)).method()` | Waits the full `ms`, *then* checks |

---

### 1. `verify(mock).method()`

> **Timestamp:** [04:40](https://www.youtube.com/watch?v=wasBpF2fhwg&t=280s) · Internal recording: [07:01](https://www.youtube.com/watch?v=wasBpF2fhwg&t=421s)

```java
class UserService {

    private final UserRepository repository;

    UserService(UserRepository repository) {
        this.repository = repository;
    }

    void register(User user) {
        repository.save(user);
    }
}
```

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    UserRepository repository;

    @InjectMocks
    UserService userService;

    @Test
    void register_savesUser() {
        User user = new User("John");

        userService.register(user);

        verify(repository).save(user);
    }
}
```

`verify(repository).save(user)` is really shorthand for `verify(repository, times(1)).save(user)` — with no count argument, Mockito defaults to "called exactly once."

Internally, this works off the same recording mechanism covered in `013 mockito architecture`: every call on a mock is intercepted (via the `MockMethodInterceptor` → `MockHandler`) and recorded into an `InvocationContainer`. `verify()` doesn't re-run anything — it simply inspects that recorded list of invocations and checks whether one matching `save(user)` happened the expected number of times.

> **Note: `verify()` only accepts a Mock or a Spy.**
>
> This follows directly from the Mockito Architecture topic — a real object (created with `new`) has no `MockMethodInterceptor` attached to it, so none of its method calls are ever intercepted or recorded into an `InvocationContainer`. With nothing recorded, there's nothing for `verify()` to check against. Only Mocks and Spies are backed by that interception machinery, so only they can be verified.

#### Added by ChatGPT — verify observable behavior

`verify()` does not call the dependency again; it checks the invocation history that the mock or spy has already recorded. Verify interactions that form part of the class's observable contract—such as saving a user or sending a notification. Verifying every internal helper call can make a test fail after a harmless refactor even though the externally visible behavior is still correct.

---

### 2. `times(n)` and `never()`

> **Timestamp:** `times(n)` [08:43](https://www.youtube.com/watch?v=wasBpF2fhwg&t=523s) · `never()` [08:55](https://www.youtube.com/watch?v=wasBpF2fhwg&t=535s)

```java
@Test
void register_callsSaveExactlyOnce() {
    User user = new User("John");

    userService.register(user);

    verify(repository, times(1)).save(user);
}
```

```java
@Test
void deleteInactiveUser_neverCallsSave() {
    User user = new User("Inactive");

    userService.deleteInactiveUser(user);

    verify(repository, never()).save(user);
}
```

- `times(n)` — asserts the call happened **exactly** `n` times. `times(1)` is what plain `verify(mock).method()` defaults to.
- `never()` — asserts the call happened **zero** times. It's just sugar for `times(0)`.

---

### 3. `atLeastOnce()`, `atLeast(n)`, `atMostOnce()`, `atMost(n)`, `only()`

> **Timestamp:** `atLeast...` [09:24](https://www.youtube.com/watch?v=wasBpF2fhwg&t=564s) · `atMost...` [10:21](https://www.youtube.com/watch?v=wasBpF2fhwg&t=621s) · `only()` [10:41](https://www.youtube.com/watch?v=wasBpF2fhwg&t=641s)

```java
@Test
void retrySave_callsSaveAtLeastOnce() {
    userService.retrySave(user);

    verify(repository, atLeastOnce()).save(user);
}

@Test
void retrySave_callsSaveAtLeastThreeTimes() {
    userService.retrySave(user);

    verify(repository, atLeast(3)).save(user);
}

@Test
void register_callsSaveAtMostOnce() {
    userService.register(user);

    verify(repository, atMostOnce()).save(user);
}

@Test
void register_callsSaveAtMostThreeTimes() {
    userService.register(user);

    verify(repository, atMost(3)).save(user);
}

@Test
void register_onlyInteractionWithRepositoryIsSave() {
    userService.register(user);

    verify(repository, only()).save(user);
}
```

- `atLeastOnce()` / `atLeast(n)` — useful for retry logic, where you know a minimum number of attempts must happen but the exact count can vary.
- `atMostOnce()` / `atMost(n)` — useful for capping calls, e.g. asserting an expensive operation isn't repeated more than expected.
- `only()` — stricter than `times(1)`. It asserts **two** things at once: this call happened exactly once, *and* it was the **only** method ever invoked on that mock. If any other method on `repository` was also called, `only()` fails — even if `save(user)` itself was called correctly.

---

### 4. Argument Matching

> **Timestamp:** [10:39](https://www.youtube.com/watch?v=wasBpF2fhwg&t=639s) · Custom validation: [12:19](https://www.youtube.com/watch?v=wasBpF2fhwg&t=739s) · Cross-parameter validation: [19:20](https://www.youtube.com/watch?v=wasBpF2fhwg&t=1160s)

Sometimes you don't care about verifying an exact value — you care about the *shape* of what was passed, or you want to capture it to assert on afterward.

**Built-in matchers:**

```java
@Test
void register_calledWithAnyUser() {
    userService.register(user);

    verify(repository).save(any(User.class));
}

@Test
void chargeCard_calledWithExactAmount() {
    paymentService.chargeCard(100);

    verify(gateway).charge(eq(100));
}
```

> **Important gotcha:** if you use a matcher like `any()` or `eq()` for **one** argument in a call, you must use matchers for **all** arguments of that call — you can't mix a raw value with a matcher in the same call. This is the same rule that applies to stubbing with `when()`, covered in `016 stubbing`.

**Custom conditions with `argThat()` / `intThat()`:**

```java
@Test
void charge_calledWithPositiveAmount() {
    paymentService.chargeCard(100);

    verify(gateway).charge(intThat(amount -> amount > 0));
}

@Test
void register_calledWithUserHavingValidEmail() {
    userService.register(user);

    verify(repository).save(argThat(u -> u.getEmail().contains("@")));
}
```

- `argThat(condition)` — for `Object`-typed parameters, takes a lambda/predicate describing what makes the argument valid.
- `intThat(condition)` — the primitive-`int` counterpart of `argThat`, since primitives can't be matched by an `Object`-based matcher directly. Similar `xThat()` variants exist for other primitive types (`longThat`, `booleanThat`, etc.).

**`ArgumentCaptor` — capturing arguments to inspect after the call:**

```java
@Test
void register_capturesUserPassedToSave() {
    ArgumentCaptor<User> captor = ArgumentCaptor.forClass(User.class);

    userService.register(new User("John", "john@mail.com"));

    verify(repository).save(captor.capture());

    User capturedUser = captor.getValue();
    assertEquals("John", capturedUser.getName());
    assertEquals("john@mail.com", capturedUser.getEmail());
}
```

Unlike matchers (which just check "does this argument satisfy X" at verification time), `ArgumentCaptor` actually **grabs the real object** that was passed in, so you can run further assertions on it — useful when the object has several fields you want to check individually, or when you want to validate relationships **across multiple captured arguments**:

```java
@Test
void chargeCard_capturesAmountAndCurrency() {
    ArgumentCaptor<Integer> amountCaptor = ArgumentCaptor.forClass(Integer.class);
    ArgumentCaptor<String> currencyCaptor = ArgumentCaptor.forClass(String.class);

    paymentService.chargeCard(100, "USD");

    verify(gateway).charge(amountCaptor.capture(), currencyCaptor.capture());

    assertEquals(100, amountCaptor.getValue());
    assertEquals("USD", currencyCaptor.getValue());
}
```

#### Added by ChatGPT — captor limitations

An `ArgumentCaptor` captures the argument reference that was passed to the mock; it does not automatically create a deep copy. If production code mutates that same object later, the captured object can reflect the mutated state. Capture immutable values where possible, or take a defensive snapshot when mutation is part of the scenario.

Captors are clearest during verification. Although `captor.capture()` can technically appear while stubbing, doing so can reduce readability and may leave the captor without a value if the stub is never invoked.

---

### 5. Order — `InOrder`

> **Timestamp:** [21:32](https://www.youtube.com/watch?v=wasBpF2fhwg&t=1292s) · Example: [24:06](https://www.youtube.com/watch?v=wasBpF2fhwg&t=1446s) · `InOrder` implementation: [25:57](https://www.youtube.com/watch?v=wasBpF2fhwg&t=1557s)

By default, `verify()` calls don't care about sequence — each one just checks its own invocation independently. When the **order of calls matters** (e.g. you must save before you notify), use `InOrder`.

**Single mock:**

```java
@Test
void testInOrderSingleMock() {
    repository.save(user);
    repository.findById(user.getId());

    InOrder inOrder = inOrder(repository);

    inOrder.verify(repository).save(user);
    inOrder.verify(repository).findById(user.getId());
}
```

**Multiple mocks:**

```java
@Test
void testInOrderMultipleMocks() {
    orderService.placeOrder(order);

    InOrder inOrder = inOrder(gateway, orderRepo, notifier);

    inOrder.verify(gateway).charge(order.getAmount());
    inOrder.verify(orderRepo).save(order);
    inOrder.verify(notifier).sendMail(order);
}
```

`inOrder(...)` takes the mocks whose relative call order you want to check, and every `inOrder.verify(...)` after that must match calls in the sequence they were declared. If the real execution order doesn't match — even if each individual call happened the right number of times — the verification fails.

#### Added by ChatGPT

`InOrder` verifies the relative order of the interactions you ask it to verify. It does not automatically mean that no other calls occurred. If extra calls must also be forbidden, finish the required ordered checks and then explicitly verify that no relevant interactions remain.

---

### 6. `verifyNoInteractions()` and `verifyNoMoreInteractions()`

> **Timestamp:** `verifyNoInteractions()` [30:25](https://www.youtube.com/watch?v=wasBpF2fhwg&t=1825s) · `verifyNoMoreInteractions()` [32:34](https://www.youtube.com/watch?v=wasBpF2fhwg&t=1954s)

```java
@Test
void testVerifyNoInteractions() {
    // repository was never touched at all in this test path

    verifyNoInteractions(repository);
}

@Test
void testVerifyNoMoreInteractions() {
    userService.register(user);

    verify(repository).save(user);

    verifyNoMoreInteractions(repository);
}
```

- **`verifyNoInteractions(mock)`** — asserts the mock was **never called at all**, for anything. Useful for confirming a branch of code correctly skips a dependency (e.g. an early-return path never reaches the repository).
- **`verifyNoMoreInteractions(mock)`** — asserts that beyond the calls you've *already* verified with `verify(...)` earlier in the test, **nothing else** happened on that mock. This catches unexpected extra calls that your explicit `verify()` statements didn't check for.

> Order matters here: `verifyNoMoreInteractions()` only looks at what's still "unverified" at the point it runs, so it should come *after* your other `verify()` calls for that mock, not before.

#### Added by ChatGPT

Use `verifyNoMoreInteractions()` selectively. It is valuable when unexpected dependency calls would be a real bug, but applying it to every mock can over-specify implementation details and make otherwise safe refactoring unnecessarily difficult.

---

### 7. `timeout(ms)` and `after(ms)` — Verifying Async Behavior

> **Timestamp:** [34:39](https://www.youtube.com/watch?v=wasBpF2fhwg&t=2079s) · Immediate-verification problem: [36:27](https://www.youtube.com/watch?v=wasBpF2fhwg&t=2187s) · `timeout()` versus `after()`: [39:00](https://www.youtube.com/watch?v=wasBpF2fhwg&t=2340s)

Both of these exist for the same problem: **the method under test kicks off work on a background thread**, and by the time your `verify()` line runs, that background work may not have finished yet — so a plain `verify()` would fail, not because the call never happens, but because it just hasn't happened *yet*.

```java
class AsyncOrderService {

    private final OrderRepository orderRepo;
    private final EmailService emailService;
    private final ExecutorService executor = Executors.newSingleThreadExecutor();

    AsyncOrderService(OrderRepository orderRepo, EmailService emailService) {
        this.orderRepo = orderRepo;
        this.emailService = emailService;
    }

    void placeOrder(Order order) {
        orderRepo.save(order); // synchronous, fast (~2ms)

        executor.submit(() -> {
            try {
                Thread.sleep(100); // simulate slow email send
            } catch (InterruptedException ignored) {
            }
            emailService.sendMail(order); // happens later, on a different thread
        });
    }
}
```

```java
@ExtendWith(MockitoExtension.class)
class AsyncOrderServiceTest {

    @Mock
    OrderRepository orderRepo;

    @Mock
    EmailService emailService;

    @InjectMocks
    AsyncOrderService asyncOrderService;

    @Test
    void testAsyncOrderService_WithTimeout() {
        Order order = new Order();

        asyncOrderService.placeOrder(order);

        verify(orderRepo, timeout(500)).save(order);
        verify(emailService, timeout(500)).sendMail(order);
    }

    @Test
    void testAsyncOrderService_WithAfter() {
        Order order = new Order();

        asyncOrderService.placeOrder(order);

        verify(emailService, after(500).times(1)).sendMail(order);
    }
}
```

**Why `timeout(500)` and not a plain `verify()`:**

`placeOrder()` returns almost immediately — `orderRepo.save(order)` takes ~2ms and runs synchronously, but `emailService.sendMail(order)` runs on a background thread after a ~100ms delay. If the test called `verify(emailService).sendMail(order)` right after `placeOrder()` returns (at ~3ms), the email call hasn't happened yet and the verification would fail — not because the code is broken, but because the test didn't wait long enough.

`timeout(500)` fixes this by **polling**: it repeatedly checks for the expected invocation, up to a maximum wait of 500ms, and returns **as soon as** the call is detected — it doesn't necessarily wait the full 500ms. This makes the test both correct and fast (it typically finishes around the ~100ms mark, not the full timeout).

#### Added by ChatGPT — making async tests reliable

Mockito's timeout modes wait for an invocation, but they do not make the background operation itself deterministic or shut down its executor. When you control the production API, returning a `CompletableFuture` or exposing another completion signal usually lets the test wait for completion directly. Also close test-created executors in teardown so their threads cannot leak into later tests.

**Why `after(500)` is different:**

`after(ms)` always waits the **full** duration before checking — there's no early exit once a matching call is seen. This makes it slower than `timeout()`, but it gives you a complete, stable picture of *how many* times the method was called during that window. That's why `after(500).times(1)` is meaningful: it's asserting "over the full 500ms window, `sendMail` was called exactly once" — something `timeout()` can't safely guarantee, since it stops polling the moment it sees the first matching call and could miss a second one arriving shortly after.

| | `timeout(ms)` | `after(ms)` |
|---|---|---|
| Waits | Up to `ms`, exits early once matched | Always the full `ms` |
| Speed | Fast (usually finishes early) | Slow (always pays the full wait) |
| Best for | "Did this eventually happen?" | "Exactly how many times did this happen, over this window?" |
| Combine with `times()`/`never()` | Riskier — may exit before a second call arrives | Safe — full window is observed before counting |

> A note on `InterruptedException` in the background thread: Java's thread interruption is **cooperative** — calling `Thread.sleep()` inside the submitted task means it *can* be interrupted, but nothing in this code forces the executor to stop early. If the test process ends before the background task completes, the task may simply never finish; `timeout()`/`after()` don't kill the background thread, they just wait (bounded) for evidence that it did its job.
