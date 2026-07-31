### `@Mock` as Method/Constructor Parameter

> **Video:** [@Mock as Method/Constructor Parameter](https://www.youtube.com/watch?v=f1wjaOF6fl4)

In the previous topic (`014 injectmock spy`), we saw the usage of `@InjectMocks` and how it works, with `MockitoExtension`:

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    PaymentGateway gateway;

    @Mock
    OrderRepository repo;

    @InjectMocks
    OrderService service;

    @Test
    void placeOrder_success() {
        Mockito.when(gateway.charge(100)).thenReturn(true);
        boolean result = service.placeOrder(100);
        assertTrue(result);
        Mockito.verify(repo).save();
    }
}
```

We've already seen how `MockitoExtension` adds hooks into our code:

- **`@BeforeEach`** — in the before-each method, resolves `@Mock` and `@InjectMocks`.
- **`@AfterEach`** — in the after-each method, closes the resources.

```java
public class MockitoExtension implements BeforeEachCallback, AfterEachCallback, ParameterResolver {
    ...
}
```

**Now, as a curious engineer, one thing might come to your mind:**

> In `MockitoExtension`, I understand the use of `BeforeEachCallback` and `AfterEachCallback`. But why does `MockitoExtension` *also* implement `ParameterResolver`?

(`ParameterResolver` was already covered in depth in the `005 extensions` topic.)

---

### Video Topic Index

- [00:00 — Introduction](https://www.youtube.com/watch?v=f1wjaOF6fl4&t=0s)
- [00:32 — Recap of field-level `@Mock` and `@InjectMocks`](https://www.youtube.com/watch?v=f1wjaOF6fl4&t=32s)
- [01:01 — `MockitoExtension` before/after callbacks](https://www.youtube.com/watch?v=f1wjaOF6fl4&t=61s)
- [03:14 — Why `MockitoExtension` also implements `ParameterResolver`](https://www.youtube.com/watch?v=f1wjaOF6fl4&t=194s)
- [03:33 — Using `@Mock` on method and constructor parameters](https://www.youtube.com/watch?v=f1wjaOF6fl4&t=213s)
- [03:58 — General JUnit `ParameterResolver` flow](https://www.youtube.com/watch?v=f1wjaOF6fl4&t=238s)
- [05:08 — `supportsParameter()` and `resolveParameter()`](https://www.youtube.com/watch?v=f1wjaOF6fl4&t=308s)
- [05:45 — How `MockitoExtension` resolves `@Mock` parameters](https://www.youtube.com/watch?v=f1wjaOF6fl4&t=345s)
- [06:18 — `@Mock` as a constructor parameter](https://www.youtube.com/watch?v=f1wjaOF6fl4&t=378s)
- [06:56 — Constructor-parameter resolution flow](https://www.youtube.com/watch?v=f1wjaOF6fl4&t=416s)
- [07:46 — Benefits of constructor-parameter mocks](https://www.youtube.com/watch?v=f1wjaOF6fl4&t=466s)
- [08:07 — Constructor-parameter limitations](https://www.youtube.com/watch?v=f1wjaOF6fl4&t=487s)
- [08:35 — Risk with the `PER_CLASS` lifecycle](https://www.youtube.com/watch?v=f1wjaOF6fl4&t=515s)
- [08:53 — Why the test constructs the service manually](https://www.youtube.com/watch?v=f1wjaOF6fl4&t=533s)
- [11:31 — `@Mock` as a test-method parameter](https://www.youtube.com/watch?v=f1wjaOF6fl4&t=691s)
- [12:00 — Method-parameter resolution flow](https://www.youtube.com/watch?v=f1wjaOF6fl4&t=720s)
- [12:48 — Method-only scope and repetition tradeoff](https://www.youtube.com/watch?v=f1wjaOF6fl4&t=768s)
- [13:42 — Both field and parameter approaches are valid](https://www.youtube.com/watch?v=f1wjaOF6fl4&t=822s)

---

### Recall How `ParameterResolver` Works

> **Video timestamp:** [03:58](https://www.youtube.com/watch?v=f1wjaOF6fl4&t=238s)

From the Extensions topic:

- JUnit tries to invoke `testMethod1`, but it requires a `Student` object.
- To resolve this, JUnit calls **each registered Extension that implements `ParameterResolver`**, and checks with each one: "can *you* create this `Student` object?"

```java
@ExtendWith(StudentParameterResolver.class)
class MyServiceTest {

    @Test
    void testMethod1(Student student) {
        System.out.println("Student name is: " + student.name);
    }
}
```

```java
public class StudentParameterResolver implements ParameterResolver {

    @Override
    public boolean supportsParameter(ParameterContext parameterContext,
                                      ExtensionContext extensionContext)
            throws ParameterResolutionException {

        return parameterContext.getParameter().getType() == Student.class;
    }

    @Override
    public Object resolveParameter(ParameterContext parameterContext,
                                    ExtensionContext extensionContext)
            throws ParameterResolutionException {

        return new Student("myName", 27);
    }
}
```

**Similarly, `MockitoExtension` implements `ParameterResolver` too** — and it supports and can resolve `@Mock`-annotated parameters.

**`MockitoExtension` framework class (conceptually):**

```java
@Override
public boolean supportsParameter(ParameterContext parameterContext, ExtensionContext context)
        throws ParameterResolutionException {

    return parameterResolver.supportsParameter(parameterContext, context);
}

@Override
@SuppressWarnings("unchecked")
public Object resolveParameter(ParameterContext parameterContext, ExtensionContext context)
        throws ParameterResolutionException {

    Object resolvedParameter = parameterResolver.resolveParameter(parameterContext, context);
    if (resolvedParameter instanceof ScopedMock) {
        context.getStore(MOCKITO).get(MOCKS, Set.class).add(resolvedParameter);
    }

    return resolvedParameter;
}
```

So yes — **we can use `@Mock` annotations as a method or constructor parameter too**, not just as a field.

---

### Used as Parameter in Constructor

> **Video timestamp:** [06:18](https://www.youtube.com/watch?v=f1wjaOF6fl4&t=378s)

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    private final PaymentGateway gateway;
    private final OrderRepository repo;
    private final OrderService service;
    /*
    Benefit:
    - Clean.
    - We can make fields 'final'.

    Cons:
    - Risky for PER_CLASS lifecycle. As the constructor is invoked
      only when the object is created.

    - @InjectMocks can not be used. As it can only resolve @Mock
      objects, not build the object under test for you.
    */

    OrderServiceTest(@Mock PaymentGateway gateway, @Mock OrderRepository repo) {

        this.gateway = gateway;
        this.repo = repo;
        this.service = new OrderService(gateway, repo);
    }

    @Test
    void placeOrder_success() {
        when(gateway.charge(100)).thenReturn(true);

        boolean result = service.placeOrder(100);

        assertTrue(result);
        verify(repo).save();
    }
}
```

---

### Used as Parameter in Method

> **Video timestamp:** [11:31](https://www.youtube.com/watch?v=f1wjaOF6fl4&t=691s)

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Test
    /*
    Mocks are created only for this method.
    */
    void placeOrder_success(@Mock PaymentGateway gateway, @Mock OrderRepository repo) {

        OrderService service = new OrderService(gateway, repo);

        when(gateway.charge(100)).thenReturn(true);

        boolean result = service.placeOrder(100);

        assertTrue(result);
        verify(repo).save();
    }
}
```

> Here, unlike constructor parameters (which apply to **every** test method in the class, since the constructor runs once per test instance), method parameters give you **mocks scoped to just that one test method** — useful when only a specific test needs a particular mock, instead of polluting the whole class with fields or a constructor.

---

### Additional Points From the Video

#### Method-Parameter Mocks Must Be Repeated Where Needed

> **Video timestamp:** [12:48](https://www.youtube.com/watch?v=f1wjaOF6fl4&t=768s)

Method parameters keep a mock local to one test, but this also means another test method cannot automatically reuse it. If many test methods need the same dependencies, each method must repeat the parameters and construct its own service:

```java
@Test
void test1(@Mock PaymentGateway gateway, @Mock OrderRepository repo) {
    OrderService service = new OrderService(gateway, repo);
    // test1...
}

@Test
void test2(@Mock PaymentGateway gateway, @Mock OrderRepository repo) {
    OrderService service = new OrderService(gateway, repo);
    // test2...
}
```

Therefore:

- use **method parameters** when only one or a few tests need those mocks;
- use **fields or constructor parameters** when most tests need the same kinds of dependencies.

#### Constructor Parameters Are Safest With the Default `PER_METHOD` Lifecycle

> **Video timestamp:** [08:35](https://www.youtube.com/watch?v=f1wjaOF6fl4&t=515s)

With JUnit's default `PER_METHOD` lifecycle, a new test-class instance is created for every test method. Its constructor runs again, so `MockitoExtension` supplies fresh constructor-parameter mocks for each test.

With `PER_CLASS`, JUnit creates only one test-class instance. Its constructor runs once, so the same constructor-created mocks are shared by all test methods. Their stubbing and invocation history can leak between tests.

---

### Explained by ChatGPT

#### Parameter Resolution vs. `@InjectMocks` Constructor Injection

These mechanisms both involve constructors, but they create different objects and run at different points:

| Mechanism | Who invokes the constructor? | Object being created | What Mockito supplies |
|---|---|---|---|
| Test constructor parameter resolution | JUnit creates the **test class** | `OrderServiceTest` | Only parameters annotated with `@Mock` |
| `@InjectMocks` constructor injection | Mockito creates the **class under test** | `OrderService` | Available mock/spy dependencies |

With constructor-parameter mocks:

```java
OrderServiceTest(@Mock PaymentGateway gateway, @Mock OrderRepository repo) {
    // JUnit is constructing OrderServiceTest.
    // MockitoExtension resolves the two @Mock parameters.
    this.service = new OrderService(gateway, repo); // We construct the SUT.
}
```

With field annotations:

```java
@Mock
PaymentGateway gateway;

@Mock
OrderRepository repo;

@InjectMocks
OrderService service; // Mockito constructs the SUT and injects the mocks.
```

This is why the constructor-parameter example manually calls `new OrderService(...)`: `ParameterResolver` resolves the annotated parameters; it does not automatically create the system under test.

#### Parameter and Field Mocks Are Separate Objects

If the same test declares both a field mock and a parameter mock of the same type, Mockito creates distinct mock instances:

```java
@Mock
PaymentGateway fieldGateway;

@Test
void test(@Mock PaymentGateway parameterGateway) {
    assertNotSame(fieldGateway, parameterGateway);
}
```

Stubbing or verifying one does not affect the other. Clear names help prevent accidentally configuring one mock while the class under test uses the other.
