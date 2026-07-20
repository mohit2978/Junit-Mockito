### What is an Extension?

> In simple terms, it allows us to add **custom behavior** to tests, like:

- Enable or disable a test conditionally.
- Add custom code before or after tests.
- Intercept method calls for logging, etc.

Remember this diagram from the "Architecture" topic:

![alt text](image.png)

**Where do Extensions fit in:**

![alt text](image-1.png)

Let's first go through each extension, then at the end we'll see the dummy flow to visualize the **ORDERING** of JUnit5 extensions — that will clarify a lot of things.

Some of the extension types are:

- Lifecycle Extensions
- Parameter Resolver
- Execution Condition
- Test Execution Callbacks (Before and After)
- Invocation Interceptor
- Exception Handler
- Instance Post Processor

---

### 1. Lifecycle Extensions

- Similar to the lifecycle annotations `@BeforeAll`, `@BeforeEach`, `@AfterAll`, `@AfterEach`.
- Used to execute custom logic before/after test classes or methods.

**Why do we need lifecycle extensions when we already have `@BeforeAll`/`@BeforeEach`/`@AfterAll`/`@AfterEach`?**

> Those lifecycle-annotated methods are tied to a **single class**. Lifecycle **extensions** are **reusable across many test classes**.

Our custom class implements the `BeforeAllCallback` extension interface:

```java
public class BeforeAllLifecycleExtension implements BeforeAllCallback {

    @Override
    public void beforeAll(ExtensionContext context) {
        System.out.println("Extension: Before all tests");
    }
}
```

Our custom class implements the `AfterAllCallback` extension interface:

```java
public class AfterAllLifecycleExtension implements AfterAllCallback {

    @Override
    public void afterAll(ExtensionContext context) throws Exception {
        System.out.println("Extension: After all tests");
    }
}
```

> The `ExtensionContext` object has all the info about: the **test class** (`getTestClass()`), the **test method** (`getTestMethod()`), **annotations** (`getElement()`), and the **test instance** (`getTestInstance()`).

**Registering multiple lifecycle extensions (comma separated) within a test class:**

```java
@ExtendWith({BeforeAllLifecycleExtension.class, AfterAllLifecycleExtension.class})
class MyServiceTest {

    MyServiceTest() {
        System.out.println("instance created");
    }

    @BeforeAll
    public static void beforeAllMethod() {
        System.out.println("inside beforeAll");
    }

    @BeforeEach
    public void beforeEachMethod() {
        System.out.println("inside beforeEach");
    }

    @Test
    void testMethod1() {
        System.out.println("inside test method");
    }

    @AfterEach
    public void afterEachMethod() {
        System.out.println("inside afterEach method");
    }

    @AfterAll
    public static void afterAllMethod() {
        System.out.println("inside afterAll method");
    }
}
```

Or we can write **one class** which implements all the lifecycle callback extension interfaces:

```java
public class LifecycleExtension implements BeforeAllCallback, AfterAllCallback,
        BeforeEachCallback, AfterEachCallback {

    @Override
    public void beforeAll(ExtensionContext context) {
        System.out.println("Extension: Before all tests");
    }

    @Override
    public void afterAll(ExtensionContext context) throws Exception {
        System.out.println("Extension: After all tests");
    }

    @Override
    public void afterEach(ExtensionContext context) throws Exception {
        System.out.println("Extension: After each tests");
    }

    @Override
    public void beforeEach(ExtensionContext context) throws Exception {
        System.out.println("Extension: Before each tests");
    }
}
```

```java
@ExtendWith(LifecycleExtension.class)
class MyServiceTest {
    // same test class as above
}
```

Output:

![alt text](image-2.png)

---

### 2. Parameter Resolver Extension

> With this, we can **dynamically supply values** to method or constructor parameters within a test class.

```java
public class Student {
    String name;
    int age;

    public Student(String name, int age) {
        this.name = name;
        this.age = age;
    }
}
```

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

- `@ExtendWith(StudentParameterResolver.class)` **at class level** — applied to all methods and constructors.
- Can also be used **at method level** — then applied to that particular method only.

> `ParameterContext` has all the info about the particular parameter: its position in the parameter list, its type, and which method or constructor declared this variable.

Output:

![alt text](image-3.png)

---

### 3. Execution Condition Extension

> Helps decide whether a test should be **enabled or disabled at runtime**, based on custom logic.

**`junit-platform.properties`:**

![alt text](image-4.png)

```java
public class ConditionalExecutionExtension implements ExecutionCondition {

    @Override
    public ConditionEvaluationResult evaluateExecutionCondition(ExtensionContext context) {

        boolean featureEnabled = Boolean.parseBoolean(
                context.getConfigurationParameter("xyz.feature.enabled").orElse("false")
        );

        return featureEnabled
                ? ConditionEvaluationResult.enabled("feature enabled, test can run")
                : ConditionEvaluationResult.disabled("feature disabled, test can not run");
    }
}
```

`@ExtendWith(ConditionalExecutionExtension.class)` used on top of a **method** — applies only to that method. Used on top of a **class** — applies to all methods of that class.

```java
class MyServiceTest {

    @Test
    @ExtendWith(ConditionalExecutionExtension.class)
    void testMethod1() {
        System.out.println("inside testMethod1");
    }

    @Test
    void testMethod2() {
        System.out.println("inside testMethod2");
    }
}
```

Output — only `testMethod1` is gated by the condition, so it's ignored while `testMethod2` runs normally:

![alt text](image-5.png)

Using the extension on top of the class instead:

```java
@ExtendWith(ConditionalExecutionExtension.class)
class MyServiceTest {

    @Test
    void testMethod1() {
        System.out.println("inside testMethod1");
    }

    @Test
    void testMethod2() {
        System.out.println("inside testMethod2");
    }
}
```

Output (feature is currently disabled — both tests fail the condition):

![alt text](image-6.png)

---

### 4. Test Execution Callback Extension

> Adds code logic **before or after** a test method executes.

We could do the same with `@BeforeEach`/`@AfterEach`, but those carry setup logic — it's not a good idea to mix them with custom logic like recording execution time, etc.

- **`BeforeTestExecutionCallback`** — runs just **before** the invocation of the test method (after `BeforeAllCallback`, `@BeforeAll`, `BeforeEachCallback`, `@BeforeEach`).
- **`AfterTestExecutionCallback`** — runs just **after** the invocation of the test method (before `@AfterEach`, `AfterEachCallback`, `@AfterAll`, `AfterAllCallback`).

```java
public class TestExecutionExtension implements BeforeTestExecutionCallback,
        AfterTestExecutionCallback {

    @Override
    public void beforeTestExecution(ExtensionContext context) {
        System.out.println("Starting test at: " + Instant.now());
    }

    @Override
    public void afterTestExecution(ExtensionContext context) {
        System.out.println("Finished test at: " + Instant.now());
    }
}
```

`@ExtendWith(TestExecutionExtension.class)` can also be used on top of a particular test method, if we don't want it applied to all test methods.

```java
@ExtendWith(TestExecutionExtension.class)
class MyServiceTest {

    @BeforeEach
    void beforeEachMethod() {
        System.out.println("inside beforeEach method");
    }

    @Test
    void testMethod1() {
        System.out.println("inside testMethod1");
    }

    @AfterEach
    void afterEachMethod() {
        System.out.println("inside afterEach method");
    }
}
```

Output:

![alt text](image-7.png)

---

### 5. Invocation Interceptor Extension

> Helps **wrap the execution** of a test method — so we can write **pre-processing** and **post-processing** logic.

```java
public class CustomInvocationExtension implements InvocationInterceptor {

    @Override
    public void interceptTestMethod(Invocation<Void> invocation,
                                     ReflectiveInvocationContext<Method> context,
                                     ExtensionContext extensionContext) throws Throwable {

        System.out.println("Pre-processing before test");

        invocation.proceed(); // actual test runs here

        System.out.println("Post-processing after test");
    }
}
```

**Used at class level:**

```java
@ExtendWith(CustomInvocationExtension.class)
class MyServiceTest {

    @Test
    void testMethod1() {
        System.out.println("inside testMethod1");
    }
}
```

![alt text](image-8.png)

**Used at method level:**

```java
class MyServiceTest {

    @Test
    void testMethod1() {
        System.out.println("inside testMethod1");
    }

    @Test
    @ExtendWith(CustomInvocationExtension.class)
    void testMethod2() {
        System.out.println("inside testMethod2");
    }
}
```

![alt text](image-9.png)

---

### 6. Exception Handler Extension

> Helps intercept exceptions thrown by a test method and handle them in our own way.

```java
public class CustomExceptionHandler implements TestExecutionExceptionHandler {

    @Override
    public void handleTestExecutionException(ExtensionContext context, Throwable throwable)
            throws Throwable {

        System.out.println("Exception occurred in test: " + context.getDisplayName());
        System.out.println("Exception message: " + throwable.getMessage());

        throw throwable;
    }
}
```

```java
@ExtendWith(CustomExceptionHandler.class)
class MyServiceTest {

    @Test
    void testMethod1() {
        System.out.println("inside testMethod1");
        int x = 1 / 0;
    }
}
```

Output:

![alt text](image-10.png)

---

### 7. Instance Post Processor Extension

> Helps **manipulate or initialize** the test instance **after it has been created** — it runs before any lifecycle method or extension runs, or even the test method itself.

```java
public class CustomPostProcessorExtension implements TestInstancePostProcessor {

    @Override
    public void postProcessTestInstance(Object testObj, ExtensionContext context) {
        if (testObj instanceof MyServiceTest) {
            MyServiceTest myServiceTestObj = (MyServiceTest) testObj;
            myServiceTestObj.value = 100;
        }
    }
}
```

`@ExtendWith(CustomPostProcessorExtension.class)` can also be used at method level (works well for `PER_METHOD` lifecycle, since it creates a new instance for each test method).

```java
@ExtendWith(CustomPostProcessorExtension.class)
public class MyServiceTest {

    public int value;

    @Test
    void testMethod1() {
        System.out.println("inside testMethod1: " + value);
    }
}
```

Output:

![alt text](image-11.png)

---

### Dummy Flow of the Jupiter Engine — Ordering of JUnit5 Extensions

![alt text](image-12.png)

```java
public void executeTestClass(Class<?> testClass) {

    // 1. run BeforeAllCallbacks extension and @BeforeAll
    for (BeforeAllCallback beforeAll : beforeAllCallbacks) {
        beforeAll.beforeAll();
    }

    // execute each test method
    for (Method testMethod : testClass.getDeclaredMethods()) {
        if (isTestMethod()) {
            Object testInstance = createTestInstance();
            executeTestMethod();
        }
    }

    // 10. AfterAllCallbacks extension and @AfterAll
    for (AfterAllCallback afterAll : afterAllCallbacks) {
        afterAll.afterAll();
    }
}

public void executeTestMethod() {

    try {
        // 2. TestInstancePostProcessor
        for (TestInstancePostProcessor pip : postProcessors) {
            pip.postProcessTestInstance();
        }

        // 3. BeforeEachCallback extension and @BeforeEach
        for (BeforeEachCallback beforeEach : beforeEachCallbacks) {
            beforeEach.beforeEach(context);
        }

        // 4. ParameterResolvers extension will run
        int argumentsCount = testMethod.getParameterCount();
        for (int i = 0; i < argumentsCount; i++) {
            Parameter param = testMethod.getParameters()[i];
            for (ParameterResolver resolver : parameterResolvers) {
                if (resolver.supportsParameter(param, context)) {
                    arguments[i] = resolver.resolveParameter(param, context);
                    break;
                }
            }
        }

        // 5. BeforeTestExecutionCallback execution will run
        for (BeforeTestExecutionCallback beforeTest : beforeTestExecutionCallbacks) {
            beforeTest.beforeTestExecution(context);
        }

        // 6. InvocationInterceptor wraps the test method
        invocationInterceptor.interceptTestMethod(() -> {
            testMethod.invoke(testInstance, arguments);
        }, testMethod, context);

        // 7. AfterTestExecutionCallback
        for (AfterTestExecutionCallback afterTest : afterTestExecutionCallbacks) {
            afterTest.afterTestExecution(context);
        }

    } catch (Throwable t) {
        // 8. TestExecutionExceptionHandler
        for (TestExecutionExceptionHandler handler : exceptionHandlers) {
            handler.handleTestExecutionException(context, t);
        }
    } finally {
        // 9. @AfterEach and AfterEachCallback extensions
        for (AfterEachCallback afterEach : afterEachCallbacks) {
            afterEach.afterEach(context);
        }
    }
}
```

**Ordering summary:**

1. `BeforeAllCallback` extensions + `@BeforeAll`
2. `TestInstancePostProcessor`
3. `BeforeEachCallback` extensions + `@BeforeEach`
4. `ParameterResolver` extensions
5. `BeforeTestExecutionCallback` extensions
6. `InvocationInterceptor` wraps the test method (`@Test` runs here)
7. `AfterTestExecutionCallback` extensions
8. `TestExecutionExceptionHandler` (only if an exception was thrown)
9. `@AfterEach` + `AfterEachCallback` extensions
10. `AfterAllCallback` extensions + `@AfterAll`
