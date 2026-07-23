### `@Mock` as Method/Constructor Parameter

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

### Recall How `ParameterResolver` Works

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
