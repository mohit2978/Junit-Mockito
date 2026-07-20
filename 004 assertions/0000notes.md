### JUnit5 Assertions

- Assertions are used to verify expected outcomes in tests.
- If an assertion fails, the test fails.

Let's go through each of these assertions one by one.

---

### 1. `assertEquals()`

- Checks if two values are equal or not.
- If expected and actual value matches, test case is considered PASS.

```java
assertEquals(expected, actual);
assertEquals(expected, actual, failureMessage);
```

> For primitive values it uses `==`, and for objects it uses `.equals()`. So, if you are comparing custom objects, then do override the `equals` and `hashCode` methods in your class — else the default `equals` method from the `Object` class is used, which checks for **reference equality** (2 different objects with the same value won't be considered same).

```java
import static org.junit.jupiter.api.Assertions.assertEquals;
import org.junit.jupiter.api.Test;

class MyServiceTest {

    @Test
    void testAddition() {

        // arrange
        MyService obj = new MyService();

        // act
        int output = obj.multiply(5, 6);

        // assert
        assertEquals(30, output, "both are not equal");
    }
}
```

`String` class overrides `equals()`, comparing the sequence of characters:

![alt text](image.png)

**Custom object without overriding `equals`:**

```java
public class MyService {
    String name;

    public MyService(String name) {
        this.name = name;
    }
}
```

```java
class MyServiceTest {

    @Test
    void testAssertEqualsForObject() {
        MyService object1 = new MyService("XYZ");
        MyService object2 = new MyService("XYZ");

        assertEquals(object1, object2);
    }
}
```

Fails, since reference equality is used by default:

![alt text](image-1.png)

**Custom object with `equals`/`hashCode` overridden:**

```java
import java.util.Objects;

public class MyService {
    String name;

    public MyService(String name) {
        this.name = name;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof MyService)) return false;

        MyService servObject = (MyService) o;
        return Objects.equals(this.name, servObject.name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(name);
    }
}
```

Now `assertEquals(object1, object2)` passes.

---

### 2. `assertNotEquals()`

- Checks if two values are **not** equal.
- If unexpected and actual value do not match, test case is considered PASS.

```java
assertNotEquals(unexpected, actual);
```

```java
@Test
void testAssertNotEqualsForObject() {
    MyService object = new MyService();

    int actualResult = object.multiply(5, 6); // 30
    assertNotEquals(11, actualResult);
}
```

---

### 3. `assertArrayEquals()`

Verifies 2 arrays contain the same elements in the same order.

```java
@Test
void testAssertArrayEquals() {
    int[] expected = {1, 2, 3};
    int[] actual = MyService.sortArray(new int[]{3, 1, 2});
    assertArrayEquals(expected, actual); // for primitive: 1==1, 2==2, 3==3
}
```

![alt text](image-2.png)

```java
@Test
void testAssertArrayEquals() {
    String[] arr1 = {"Hello", "World"};
    String[] arr2 = {new String("Hello"), new String("World")};

    assertArrayEquals(arr1, arr2); // for object: "hi".equals("hi"), "bye".equals("bye")
}
```

![alt text](image-3.png)

---

### 4. `assertIterableEquals()`

- Similar to `assertArrayEquals`, but used to compare two `Iterable` collections like `List`, `Queue`, etc.
- Checks element by element in order using `.equals()`.
- Iteration sequence should match.
- `Set` does not guarantee insertion order, so it's **not recommended** to use this assertion on such unordered collections.

```java
@Test
void testAssertIterableEquals() {
    List<String> expected = List.of("A", "B", "C");
    List<String> actual = List.of("A", "B", "C");

    assertIterableEquals(expected, actual);
}
```

---

### 5. `assertLinesMatch()`

- Both expected and actual must be `List<String>`.
- Each element of `List<String>` is treated as one line.
- Supports regex match too.

```java
assertLinesMatch(List<String> expectedLines, List<String> actualLines)
```

```java
@Test
void testAssertLinesMatch() {
    List<String> expected = List.of("Hello", "World");
    List<String> actual   = List.of("Hello", "World");

    assertLinesMatch(expected, actual);
}
```

**Regex matches:**

```java
@Test
void testAssertLinesMatch() {
    List<String> expected = List.of("Hello", "World from [A-Za-z]+");
    List<String> actual   = List.of("Hello", "World from XYZ");

    assertLinesMatch(expected, actual);
}
```

![alt text](image-4.png)

**Regex Patterns:**

| Pattern | Meaning |
|---|---|
| `\d+` | Digits (e.g., `"123"`, `"42"`) |
| `[A-Za-z]+` | Letters/words (e.g., `"Hello"`, `"World"`) |
| `\w+` | Alphanumeric (same as `[A-Za-z0-9_]`, includes underscore) |
| `.*` | Anything (any characters) — matches whole line, useful if only part matters |

**Regex Mismatch:**

```java
@Test
void testAssertLinesMatch() {
    List<String> expected = List.of("Hello", "World from [A-Za-z]+");
    List<String> actual   = List.of("Hello", "World from 42");

    assertLinesMatch(expected, actual);
}
```

Fails — `42` doesn't match the letters-only pattern `[A-Za-z]+`:

![alt text](image-5.png)

---

### 6. `assertSame()`

Just checks whether both objects point to the **same reference** or not.

```java
@Test
void testAssertSame() {
    String object1 = "JUnit";
    String object2 = "JUnit";   // same String literal → same reference (string pool)

    assertSame(object1, object2);
}
```

![alt text](image-6.png)

In the example below, both `object1` and `object2` point to **different** objects:

```java
@Test
void testAssertSame() {
    String object1 = "JUnit";
    String object2 = new String("JUnit");

    assertSame(object1, object2);
}
```

![alt text](image-7.png)

Fails, since `new String("JUnit")` creates a separate object:

![alt text](image-8.png)

---

### 7. `assertNotSame()`

Checks if both objects have **different** references.

```java
@Test
void testAssertNotSame() {
    String object1 = "JUnit";
    String object2 = new String("JUnit");

    assertNotSame(object1, object2);
}
```

![alt text](image-9.png)

---

### 8. `assertNull()`

Checks if the object value is `null`.

```java
@Test
void testAssertNull() {
    MyService object = null;
    assertNull(object);
}
```

![alt text](image-10.png)

---

### 9. `assertNotNull()`

Checks if the object value is **not** `null`.

```java
@Test
void testAssertNotNull() {
    MyService object = new MyService();
    assertNotNull(object);
}
```

![alt text](image-11.png)

---

### 10. `assertTrue()`

Checks if the condition is `true`.

```java
@Test
void testAssertTrue() {
    assertTrue(5 > 2);
}
```

![alt text](image-12.png)

---

### 11. `assertFalse()`

Checks if the condition is `false`.

```java
@Test
void testAssertFalse() {
    assertFalse(5 < 2);
}
```

![alt text](image-13.png)

---

### 12. `fail()`

- Fails the test deliberately.
- Generally used when the flow reaches a point where it should not.

```java
@Test
void testAssertFail() {
    MyService object = new MyService();
    try {
        object.methodThrowsException();
        fail("above method should throw exception");
    } catch (Exception e) {
        // expecting the control should reach here.
    }
}
```

![alt text](image-14.png)

If the exception is **not** thrown, `fail()` executes and the test fails with our custom message:

![alt text](image-15.png)

---

### 13. `assertInstanceOf()`

Checks if an object is an instance of the specified class (or subclass).

```java
@Test
void testAssertInstanceOf() {
    Number n = Integer.valueOf(10);

    // passes, because Integer is a subclass of Number
    assertInstanceOf(Number.class, n);

    // fails, because n is not a Double
    assertInstanceOf(Double.class, n);
}
```

![alt text](image-16.png)

---

### 14. `assertThrows()`

- If the method throws the expected exception (or any of its subclasses), the test case passes.
- Since JUnit needs to catch the exception and then compare, it expects the 2nd parameter as an `Executable`.

```java
assertThrows(Class<T> expectedType, Executable executable)
```

```java
@Test
void testAssertThrows() {
    assertThrows(Exception.class, () -> Integer.parseInt("abc"));
}
```

Passes — the exception thrown (`NumberFormatException`) is a subclass of the expected `Exception`:

![alt text](image-17.png)

Or we can directly specify the exact exception type:

```java
@Test
void testAssertThrows() {
    assertThrows(NumberFormatException.class, () -> Integer.parseInt("abc"));
}
```

In the test below, the expected exception is `NullPointerException`, but the actual exception thrown is different — hence the test case fails:

```java
@Test
void testAssertThrows() {
    assertThrows(NullPointerException.class, () -> Integer.parseInt("abc"));
}
```

![alt text](image-18.png)

---

### 15. `assertThrowsExactly()`

Must throw the exact same type of exception (**no subclasses**).

We need to provide the exact exception that's going to be thrown. If we use the parent class type, the test fails, since the exception thrown is its child, `NumberFormatException`, and it expects the exact same type:

```java
@Test
void testAssertThrowsExactly() {
    assertThrowsExactly(Exception.class, () -> Integer.parseInt("abc"));
}
```

![alt text](image-19.png)

If we provide the exact same exception, it passes:

```java
@Test
void testAssertThrowsExactly() {
    assertThrowsExactly(NumberFormatException.class, () -> Integer.parseInt("abc"));
}
```

![alt text](image-20.png)

---

### 16. `assertDoesNotThrow()`

Expectation is that the method does **not** throw any exception. If any exception is thrown, the test case will fail.

```java
@Test
void testAssertDoNotThrow() {
    assertDoesNotThrow(() -> Integer.parseInt("123"));
}
```

![alt text](image-21.png)

```java
@Test
void testAssertDoNotThrow() {
    assertDoesNotThrow(() -> Integer.parseInt("abc"));
}
```

Fails, since `"abc"` throws a `NumberFormatException`:

![alt text](image-22.png)

---

### 17. `assertTimeout()`

Ensures code finishes within time, else test fails (**but still runs to end**).

```java
assertTimeout(Duration timeout, Executable executable)
```

```java
@Test
void testAssertTimeout() {
    assertTimeout(Duration.ofMillis(100), () -> Thread.sleep(50));
}
```

![alt text](image-23.png)

The test case takes more time than expected, so it fails:

```java
@Test
void testAssertTimeout() {
    assertTimeout(Duration.ofMillis(100), () -> Thread.sleep(150));
}
```

![alt text](image-24.png)

---

### 18. `assertTimeoutPreemptively()`

- Similar to `assertTimeout()`, but aborts execution **immediately** when the timeout is exceeded.
- Internally it uses a different thread for execution, so when the timeout exceeds, it interrupts it.

---

### 19. `assertAll()`

Groups multiple assertions, runs **all** of them, and reports failures together.

```java
assertAll(Executable...executables)
```

```java
@Test
void testAssertAll() {
    String str = "JUnit5";
    assertAll(
            () -> assertEquals(6, str.length()),
            () -> assertTrue(str.startsWith("J")),
            () -> assertFalse(str.isEmpty())
    );
}
```

![alt text](image-25.png)

It properly shows the result of **each** assertion — even if multiple fail together:

```java
@Test
void testAssertAll() {
    String str = "";
    assertAll(
            () -> assertEquals(6, str.length()),
            () -> assertTrue(str.startsWith("J")),
            () -> assertFalse(str.isEmpty())
    );
}
```

![alt text](image-26.png)

Output — reports all 3 failures together (`MultipleFailuresError`), instead of stopping at the first one:

![alt text](image-27.png)
