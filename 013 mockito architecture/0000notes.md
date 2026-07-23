### Mockito Internals

```java
class Calculator {

    int add(int a, int b) {
        return a + b;
    }

    int multiply(int a, int b) {
        return a * b;
    }
}
```

```java
@Test
void testMultiply() {

    Calculator calculator = mock(Calculator.class);

    when(calculator.multiply(4, 2)).thenReturn(100);

    int output = calculator.multiply(4, 2);

    assertEquals(100, output);
}
```

---

### 1. What Happens Internally When We Create a Mock or Spy Object

```
Calculator calculator = Mockito.mock(Calculator.class);     Calculator calculator = Mockito.spy(new Calculator());
```

Both go through the same `MockMaker` interface (`<<interface>>`), which has two implementations:

![alt text](image.png)

- **`InlineMockMaker`**
  - It updates the **bytecode of the `Calculator` class itself**.
  - Adds an interceptor: `MockMethodInterceptor`.
  - Even a **final class** can be mocked.
  - This is the **default `MockMaker` in the latest Mockito version**.

- **`SubclassMockMaker`**
  - It **creates a subclass** of `Calculator`.
  - Adds the same interceptor: `MockMethodInterceptor`.
  - We **can not** mock a `final` class (since we can't create a subclass of it).

Both then flow into: `MockMethodInterceptor` → `MockHandler` → `InvocationContainer`.

> **`MockHandler`** is the **real brain of Mockito** — every method call on a mock or spy reaches here (whether it's a stub setup, a `verify()` call, or normal execution).
>
> **`InvocationContainer`** stores the stub information and the method invocation history (which method was called, how many times, with what arguments, etc.). `MockHandler` makes use of it.

> `ByteBuddy` is the underlying agent/library that actually provides the bytecode-manipulation implementation for both `InlineMockMaker` and `SubclassMockMaker`.

---

### Conceptual Code — What `InlineMockMaker` Might Have Done

Since it rewrites the bytecode of the `Calculator` class itself, conceptually the modified class looks something like this:

```java
class Calculator {

    MockMethodInterceptor mockitoInterceptor;

    void setMockitoInterceptor(MockMethodInterceptor interceptor) {
        this.mockitoInterceptor = interceptor;
    }

    int multiply(int a, int b) {

        // if interception toggle is off, run real code, it is
        // required to avoid infinite loop.
        if (interceptFlagDisabled()) {
            return a * b;
        }

        return mockitoInterceptor.doIntercept(this, multiply, new Object[]{a, b}, realMethod);
    }

    // similar changes for add method
}
```

**Use case — why `interceptFlagDisabled()` exists:**

This `interceptFlagDisabled` toggle is specifically required for the **Spy** use case, where we want to call the **real method** when stubbing isn't available for it.

```
Unit test → multiply(a, b) → intercept → MockHandler → no stubbing found →
Spy object, so by default it will invoke real method → Now interception should
be disabled, else it will cause an infinite loop.
```

(Without disabling interception, calling the real method from inside the interceptor would re-trigger the interceptor again — an infinite loop.)

---

### Conceptual Code — What `SubclassMockMaker` Might Have Done

Since it creates a subclass of `Calculator` instead of rewriting its bytecode:

```java
class CalculatorSpyProxy extends Calculator {

    MockMethodInterceptor mockitoInterceptor;

    void setMockitoInterceptor(MockMethodInterceptor interceptor) {
        this.mockitoInterceptor = interceptor;
    }

    @Override
    int add(int a, int b) {
        return mockitoInterceptor.doIntercept(this, add, new Object[]{a, b}, realMethod);
    }

    @Override
    int multiply(int a, int b) {
        return mockitoInterceptor.doIntercept(this, multiply, new Object[]{a, b}, () -> super.multiply(a, b));
    }
}
```

Here, since it's a real subclass, calling the real method is simply `super.multiply(a, b)` — no toggle flag is needed the way `InlineMockMaker` needs one, because overriding + `super` naturally avoids re-triggering the same overridden method.

---

### What Happens Internally When We Do Stubbing on Mock/Spy Objects

For `Mockito.when(calculator.multiply(4, 2)).thenReturn(100)`:

**Step 1 — `calculator.multiply(4, 2)` needs to be resolved first:**

```
Calculator calculator = Mockito.mock(Calculator.class);
Mockito.when(calculator.multiply(4, 2)).thenReturn(100);
```

```
Call reaches Calculator.multiply() method
        ↓
MockMethodInterceptor
        ↓
MockHandler
```

(In the latest Mockito version, `InlineMockMaker` is the default, so the call is intercepted via the rewritten bytecode shown above.)

**Step 2 — Within `MockHandler`, the flow is:**

![alt text](image-1.png)

```
Is Stub Present in Temp area?
  → Yes: Store the stub details in InvocationContainer, and return null.
          (Generally doReturn(...) reaches here:
           Mockito.doReturn(100).when(calculator).multiply(4, 2);
           doReturn(...).when(...) already stores the stub details in some temp area.)

Is Verify?
  → Yes: Use InvocationContainer.
          (InvocationContainer stores all the info, like which method — e.g. multiply(4, 2) —
           was invoked, how many times, etc.)

Is Stub present in InvocationContainer?
  → Yes: Update InvocationContainer (increments invocation count), and return the stub value.

Is Mock or Spy?
  → Mock: Return default values — null, false, 0 (depending on return type).
  → Spy: Invoke the real method.
          - For InlineMockMaker: interceptFlagDisabled = true, then call real code.
          - For SubclassMockMaker: invokes the parent class's method (super.<method>()).
```

![alt text](image-2.png)

**So for `Mockito.when(calculator.multiply(4, 2)).thenReturn(100)`:**

At the moment `calculator.multiply(4, 2)` is evaluated (before `.thenReturn(100)` even runs), there's no stub registered yet — and since `calculator` is a plain **Mock** (not a Spy), the flow falls through to "Is Mock or Spy? → Mock → Return default values" — so this inner call **returns `0`** (default value for `int`). *Then* `.thenReturn(100)` registers `100` as the stub value for that method+arguments combination.

---

### What Happens When the Method Is Actually Invoked

```java
Calculator calculator = Mockito.mock(Calculator.class);
Mockito.when(calculator.multiply(4, 2)).thenReturn(100);

int product = calculator.multiply(4, 2);
```

**Step 1 — the call is intercepted the same way as before:**

![alt text](image-3.png)

```
Call reaches Calculator.multiply() method
        ↓
MockMethodInterceptor
        ↓
MockHandler
```

**Step 2 — within `MockHandler`, the same decision tree runs again:**

![alt text](image-4.png)

This time:

```
Is Stub Present in Temp area?  → No
Is Verify?                     → No
Is Stub present in InvocationContainer?  → Yes (we registered it via thenReturn(100) earlier)
        → Update InvocationContainer (increments invocation count), and return the stub value.
```

![alt text](image-5.png)

**So `calculator.multiply(4, 2)` returns `100`** — the stubbed value we set up earlier.
