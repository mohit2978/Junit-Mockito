
### What are Assertions?

> Assertions are **checks** you put in your test to verify that your code produced the **expected result**.
>
> If the assertion **passes** → test is GREEN ✅
> If the assertion **fails** → test is RED ❌ with a clear error message

```java
// Basic structure:
assertThat(ACTUAL).isEqualTo(EXPECTED);
//          ↑           ↑
//    what your     what you
//    code gave     expected
```

---

### Setup — class used in all examples

```java
// Production class being tested
public class User {
    private String name;
    private int age;
    private String email;
    private boolean active;
    private List<String> roles;

    // constructor, getters, setters
    public boolean isAdult() { return age >= 18; }
}
```

---

### 1. `assertEquals` / `isEqualTo` — check exact value

```java
@Test
void shouldReturnCorrectUserName() {

    // Arrange
    User user = new User("Alice", 25, "alice@gmail.com");

    // Act
    String name = user.getName();

    // Assert
    assertEquals("Alice", name);              // JUnit style
    assertThat(name).isEqualTo("Alice");      // AssertJ style ✅ preferred

    // Failure message if wrong:
    // expected: "Alice" but was: "Bob"
}
```


---

### The Core Rule of `assertEquals`

```
Primitive types  (int, double, boolean) → uses  ==
Object types     (String, User, Order)  → uses  .equals()
```

---

### For Primitives — uses `==`

```java
@Test
void shouldAssertPrimitiveValues() {

    int a = 5;
    int b = 5;

    assertEquals(a, b);  // uses == internally → passes ✅
    // 5 == 5 → true ✅

    double price1 = 100.0;
    double price2 = 100.0;
    assertEquals(price1, price2); // 100.0 == 100.0 → true ✅

    boolean flag = true;
    assertEquals(true, flag);     // true == true → true ✅
}
```

---

### For Objects — uses `.equals()`

#### String — works fine because String OVERRIDES `.equals()`

```java
@Test
void testAssertEqualsForObject() {

    // This is the exact code from the image
    assertEquals("hello", new String("hello"));

    // Internally JUnit does:
    // "hello".equals(new String("hello"))
    // String.equals() compares CHARACTER BY CHARACTER
    // "hello" == "hello" → TRUE ✅ (content is same)

    // Even though they are TWO different objects in memory:
    // Object 1: "hello" at address 0x001
    // Object 2: new String("hello") at address 0x002
    // String.equals() doesn't care about address — compares content ✅
}
```

Why does String work? Because Java's `String` class **overrides** `equals()`:

```java
// Inside Java's String class — simplified:
public class String {

    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (obj instanceof String other) {
            // compares character by character
            return this.value.equals(other.value); // content check ✅
        }
        return false;
    }
}

// So "hello".equals(new String("hello"))
// → compares 'h','e','l','l','o'  vs  'h','e','l','l','o'
// → TRUE ✅
```

---

### Custom Object WITHOUT `equals()` override — FAILS ❌

```java
// Production class — NO equals() override
public class User {
    private String name;
    private int age;

    public User(String name, int age) {
        this.name = name;
        this.age  = age;
    }
    // NO equals() or hashCode() — uses Object's default!
}
```

```java
@Test
void shouldFailBecauseNoEqualsOverride() {

    // Arrange
    User user1 = new User("Alice", 25); // address: 0x001
    User user2 = new User("Alice", 25); // address: 0x002

    // Assert
    assertEquals(user1, user2); // ❌ FAILS!

    // Internally JUnit does:
    // user1.equals(user2)
    // → uses Object.equals() (default — NOT overridden)
    // → Object.equals() checks REFERENCE (memory address)
    // → 0x001 == 0x002 → FALSE ❌

    // Error message:
    // expected: User@1a2b3c but was: User@4d5e6f
    //           ↑                        ↑
    //      memory address           different address
}
```

---

### Custom Object WITH `equals()` override — PASSES ✅

```java
// Production class — WITH equals() and hashCode()
public class User {
    private String name;
    private int age;

    public User(String name, int age) {
        this.name = name;
        this.age  = age;
    }

    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (!(obj instanceof User other)) return false;
        // compares CONTENT — not memory address
        return Objects.equals(this.name, other.name)
            && this.age == other.age;
    }

    @Override
    public int hashCode() {
        // ALWAYS override hashCode when you override equals!
        return Objects.hash(name, age);
    }
}
```

```java
@Test
void shouldPassBecauseEqualsIsOverridden() {

    // Arrange
    User user1 = new User("Alice", 25); // address: 0x001
    User user2 = new User("Alice", 25); // address: 0x002

    // Assert
    assertEquals(user1, user2); // ✅ PASSES!

    // Internally JUnit does:
    // user1.equals(user2)
    // → uses YOUR overridden equals()
    // → compares name: "Alice" == "Alice" ✅
    // → compares age:  25     == 25      ✅
    // → returns TRUE ✅
}
```

---

### Using Lombok — easiest way to add equals/hashCode

```java
// With Lombok — auto-generates equals() and hashCode()
@Data           // generates equals, hashCode, toString, getters, setters
public class User {
    private String name;
    private int age;
    private String email;
}

// OR more specific:
@EqualsAndHashCode  // only generates equals and hashCode
public class User {
    private String name;
    private int age;
}
```

```java
@Test
void shouldWorkWithLombok() {

    User user1 = new User("Alice", 25);
    User user2 = new User("Alice", 25);

    assertEquals(user1, user2); // ✅ Lombok handles it!
}
```

---

### `==` vs `.equals()` — the full picture

```java
@Test
void shouldUnderstandDifference() {

    String s1 = "hello";              // String pool
    String s2 = "hello";              // same pool reference
    String s3 = new String("hello");  // new heap object

    // == checks REFERENCE (memory address)
    assertTrue(s1 == s2);    // ✅ same pool reference
    assertFalse(s1 == s3);   // ❌ different objects

    // .equals() checks CONTENT
    assertTrue(s1.equals(s2)); // ✅ same content
    assertTrue(s1.equals(s3)); // ✅ same content — even different objects!

    // assertEquals uses .equals() for objects:
    assertEquals(s1, s3);  // ✅ PASSES — content is same
    assertSame(s1, s3);    // ❌ FAILS — different references
    //  ↑
    // assertSame checks == (reference equality)
    // assertEquals checks .equals() (content equality)
}
```

---

### `assertEquals` vs `assertSame`

```java
@Test
void shouldShowDifferenceBetweenEqualsAndSame() {

    User user1 = new User("Alice", 25);
    User user2 = new User("Alice", 25); // same content, different object
    User user3 = user1;                 // same reference!

    // assertEquals → uses .equals() → content check
    assertEquals(user1, user2); // ✅ if equals() overridden
    assertEquals(user1, user3); // ✅ same content

    // assertSame → uses == → reference check
    assertSame(user1, user3);   // ✅ same object reference
    assertSame(user1, user2);   // ❌ different objects in memory!
}
```

---

### Why override `hashCode` too?

```java
// Rule: if two objects are equal → they MUST have same hashCode
// If you override equals but NOT hashCode:

User u1 = new User("Alice", 25);
User u2 = new User("Alice", 25);

u1.equals(u2) // → true  (equals overridden)
u1.hashCode() // → 12345 (random — NOT overridden)
u2.hashCode() // → 67890 (different random)

// This BREAKS HashMap and HashSet!
Map<User, String> map = new HashMap<>();
map.put(u1, "value");
map.get(u2); // → null! ❌ can't find it — different hashCode
             // even though u1.equals(u2) is true!

// Always override BOTH together! ✅
```

---

### Summary table

| Comparison | Uses | Works correctly without override? |
|---|---|---|
| `int`, `double`, `boolean` (primitives) | `==` | ✅ Yes |
| `String` | `.equals()` | ✅ Yes — String overrides it |
| `Integer`, `Long` (wrappers) | `.equals()` | ✅ Yes — they override it |
| Your custom `User`, `Order` class | `.equals()` | ❌ No — must override! |
| `assertSame()` | `==` always | N/A — checks reference |
| `assertEquals()` | `==` for primitives, `.equals()` for objects | Only if equals() overridden |

---

### Bottom line

> `assertEquals` uses `==` for primitives and `.equals()` for objects
>
> `String` works out of the box because Java already overrides `.equals()` to compare characters
>
> For your **custom classes** — always override **both `equals()` AND `hashCode()`** — otherwise two objects with same data will NOT be considered equal in tests, HashMaps, or HashSets
>
> Use **Lombok's `@Data` or `@EqualsAndHashCode`** to generate them automatically and avoid bugs
---

### 2. `assertNotEquals` / `isNotEqualTo` — check value is different

```java
@Test
void shouldGenerateUniqueOrderIds() {

    // Arrange
    OrderService orderService = new OrderService();

    // Act
    String id1 = orderService.generateId();
    String id2 = orderService.generateId();

    // Assert
    assertNotEquals(id1, id2);               // JUnit style
    assertThat(id1).isNotEqualTo(id2);       // AssertJ style ✅

    // Failure message if wrong:
    // expected: not equal to "ORD-001" but was: "ORD-001"
}
```

---

### 3. `assertTrue` / `isTrue` — check boolean is true

```java
@Test
void shouldConfirmUserIsAdult() {

    // Arrange
    User user = new User("Alice", 20);

    // Act
    boolean result = user.isAdult();

    // Assert
    assertTrue(result);              // JUnit style
    assertThat(result).isTrue();     // AssertJ style ✅

    // Failure message if wrong:
    // expected: true but was: false
}
```

---

### 4. `assertFalse` / `isFalse` — check boolean is false

```java
@Test
void shouldConfirmMinorIsNotAdult() {

    // Arrange
    User user = new User("Bob", 15);

    // Act
    boolean result = user.isAdult();

    // Assert
    assertFalse(result);             // JUnit style
    assertThat(result).isFalse();    // AssertJ style ✅

    // Failure message if wrong:
    // expected: false but was: true
}
```

---

### 5. `assertNull` / `isNull` — check value is null

```java
@Test
void shouldReturnNullWhenUserNotFound() {

    // Arrange
    UserService userService = new UserService();

    // Act
    User user = userService.findByEmail("unknown@gmail.com");

    // Assert
    assertNull(user);                // JUnit style
    assertThat(user).isNull();       // AssertJ style ✅

    // Failure message if wrong:
    // expected: null but was: User{name='Alice'}
}
```

---

### 6. `assertNotNull` / `isNotNull` — check value is NOT null

```java
@Test
void shouldReturnUserWhenFound() {

    // Arrange
    UserService userService = new UserService();

    // Act
    User user = userService.findByEmail("alice@gmail.com");

    // Assert
    assertNotNull(user);             // JUnit style
    assertThat(user).isNotNull();    // AssertJ style ✅

    // Failure message if wrong:
    // expected: not null but was: null
}
```

---

### 7. `assertThrows` / `assertThatThrownBy` — check exception is thrown


![alt text](image-1.png)


Great diagram! This explains the **internals of `assertThrows()`**. Let me break down every detail.

---

### The Method Signature

```java
assertThrows(Class<T> expectedType, Executable executable)
//            ↑                      ↑
//    exception class you expect    lambda / code that should throw
```

---

### Why `Executable` as second parameter?

This is the KEY insight from the diagram.

JUnit needs to:
1. **Run** your code
2. **Catch** the exception itself
3. **Compare** the caught exception with expected type

If you just called the method directly — exception would propagate to JUnit and **crash the test** before any comparison happens:

```java
// ❌ If assertThrows worked like this — WRONG design
assertThrows(
    IllegalArgumentException.class,
    service.createUser("Alice", -1) // exception thrown HERE immediately!
    // JUnit never gets to compare — test just crashes!
);

// ✅ With Executable — JUnit controls WHEN code runs
assertThrows(
    IllegalArgumentException.class,
    () -> service.createUser("Alice", -1) // wrapped in lambda
    // JUnit calls this lambda INSIDE a try-catch
    // catches exception → compares → reports result
);
```

---

### What is `Executable`?

```java
// Executable is a JUnit functional interface:
@FunctionalInterface
public interface Executable {
    void execute() throws Throwable;
}

// ANY of these satisfy Executable:

// 1. Lambda expression
() -> service.createUser("Alice", -1)

// 2. Lambda with multiple lines
() -> {
    service.validate(input);
    service.createUser("Alice", -1);
}

// 3. Method reference
() -> service.throwingMethod()
// or
service::throwingMethod

// 4. Anonymous class (old style)
new Executable() {
    @Override
    public void execute() {
        service.createUser("Alice", -1);
    }
}
```

---

### How JUnit implements `assertThrows` internally

```java
// Simplified internal implementation:
public static <T extends Throwable> T assertThrows(
        Class<T> expectedType,
        Executable executable) {

    try {
        // Step 1: JUnit CALLS your executable (lambda)
        executable.execute();

        // Step 2: If we reach here — NO exception was thrown!
        // That means test should FAIL
        fail("Expected " + expectedType.getName()
           + " to be thrown, but nothing was thrown");

    } catch (Throwable actualException) {

        // Step 3: Exception WAS thrown — check the type
        if (expectedType.isInstance(actualException)) {

            // Step 4: Type matches (or subclass) — TEST PASSES ✅
            return expectedType.cast(actualException);
            // returns the exception so you can verify details

        } else {

            // Step 5: Wrong exception type — TEST FAILS ❌
            fail("Expected " + expectedType.getName()
               + " but got " + actualException.getClass().getName());
        }
    }
}
```

Step by step flow:

```
JUnit calls executable.execute()
            │
            ▼
    Did exception occur?
            │
     ┌──────┴──────┐
    YES            NO
     │              │
     ▼              ▼
Is it expected    FAIL ❌
   type?          "Nothing was thrown"
     │
  ┌──┴──┐
 YES    NO
  │      │
  ▼      ▼
PASS ✅  FAIL ❌
Return   "Wrong exception
exception  type thrown"
```

---

### Subclass behavior — `isInstance()` check

The diagram says:
> **"If method throws expected exception OR any of its subclass — test passes"**

This is because internally JUnit uses `isInstance()` not `==`:

```java
// Internally JUnit does:
expectedType.isInstance(actualException)
// NOT:
// expectedType == actualException.getClass()

// isInstance() returns true for:
// - exact type match
// - subclass match  ← this is the key!
```

```java
// Exception hierarchy:
RuntimeException
    └── IllegalArgumentException
            └── InvalidAgeException  ← your custom exception

// Test:
assertThrows(RuntimeException.class,
    () -> service.createUser("Alice", -1)
    // throws InvalidAgeException
);

// Internally:
// RuntimeException.isInstance(new InvalidAgeException())
// → true! Because InvalidAgeException IS-A RuntimeException
// → TEST PASSES ✅
```

```java
// All 3 pass for same thrown exception (InvalidAgeException):
assertThrows(InvalidAgeException.class,   () -> ...); // ✅ exact
assertThrows(IllegalArgumentException.class,() -> ...);// ✅ parent
assertThrows(RuntimeException.class,      () -> ...); // ✅ grandparent
assertThrows(Exception.class,             () -> ...); // ✅ great-grandparent

// Only assertThrowsExactly is strict:
assertThrowsExactly(InvalidAgeException.class,    () -> ...); // ✅ exact only
assertThrowsExactly(IllegalArgumentException.class,() -> ...); // ❌ not exact!
```

---

### Complete working example

```java
// Production code
@Service
public class UserService {

    public User createUser(String name, int age) {
        if (age < 0) {
            throw new IllegalArgumentException(
                "Age cannot be negative: " + age
            );
        }
        return new User(name, age);
    }
}

// Test
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @InjectMocks
    UserService userService;

    @Test
    void shouldThrowWhenAgeIsNegative() {

        // Step 1: expectedType = IllegalArgumentException.class
        // Step 2: executable = () -> userService.createUser("Alice",-1)

        IllegalArgumentException ex =
            assertThrows(

                IllegalArgumentException.class,
                // ↑ Class<T> expectedType

                () -> userService.createUser("Alice", -1)
                // ↑ Executable executable
                // JUnit wraps this in try-catch internally
            );

        // Step 3: JUnit called executable.execute()
        // Step 4: IllegalArgumentException was thrown
        // Step 5: isInstance check passed ✅
        // Step 6: exception returned to us

        // Now verify exception details:
        assertThat(ex.getMessage())
            .isEqualTo("Age cannot be negative: -1");
    }
}
```

---

### Summary — why this design is brilliant

```
Problem:
  "How does JUnit test that MY code throws an exception
   without the exception crashing the test itself?"

Solution — Executable interface:
  1. Wrap your code in a lambda (Executable)
  2. JUnit calls the lambda inside its OWN try-catch
  3. JUnit catches the exception safely
  4. JUnit compares using isInstance() — allows subclasses
  5. JUnit returns exception — you verify details

Result:
  ✅ Exception doesn't crash the test
  ✅ JUnit controls execution
  ✅ Subclasses accepted (isInstance not ==)
  ✅ Exception returned for further verification
```

---

### Bottom line

> `assertThrows(Class<T> expectedType, Executable executable)` takes an **Executable** (lambda) as second parameter because JUnit needs to **control when your code runs** — it wraps it in its own `try-catch` to safely catch, compare, and return the exception
>
> It uses **`isInstance()`** internally — which means **subclasses of the expected exception also pass** — use `assertThrowsExactly()` if you need strict type matching with no subclasses allowed

```java
@Test
void shouldThrowExceptionWhenUserIdIsInvalid() {

    // Arrange
    UserService userService = new UserService();

    // Act + Assert — JUnit style
    assertThrows(UserNotFoundException.class,
        () -> userService.findById(-1L)
    );

    // AssertJ style ✅ — also checks message!
    assertThatThrownBy(() -> userService.findById(-1L))
        .isInstanceOf(UserNotFoundException.class)
        .hasMessage("User not found with id: -1")
        .hasMessageContaining("-1");

    // Failure message if wrong:
    // expected UserNotFoundException but no exception was thrown
}
```

---

### 8. `assertThat().contains()` — check collection contains element

```java
@Test
void shouldContainAdminRole() {

    // Arrange
    User user = new User("Alice");
    user.addRole("ADMIN");
    user.addRole("USER");

    // Act
    List<String> roles = user.getRoles();

    // Assert
    assertThat(roles).contains("ADMIN");             // contains at least this
    assertThat(roles).containsExactly("ADMIN","USER");// exact elements, exact order
    assertThat(roles).containsExactlyInAnyOrder("USER","ADMIN"); // any order
    assertThat(roles).doesNotContain("SUPER_ADMIN");  // NOT in list

    // Failure message if wrong:
    // expected to contain: "ADMIN" but did not
}
```

---

### 9. `assertThat().hasSize()` — check collection size

```java
@Test
void shouldReturnAllActiveUsers() {

    // Arrange
    UserService userService = new UserService();
    userService.save(new User("Alice", true));
    userService.save(new User("Bob",   true));
    userService.save(new User("Carol", false)); // inactive

    // Act
    List<User> activeUsers = userService.getActiveUsers();

    // Assert
    assertThat(activeUsers).hasSize(2);       // exactly 2
    assertThat(activeUsers).isNotEmpty();      // at least 1
    assertThat(activeUsers).hasSizeGreaterThan(1);
    assertThat(activeUsers).hasSizeLessThan(5);

    // Failure message if wrong:
    // expected size: 2 but was: 3
}
```

---

### 10. `assertThat().extracting()` — extract field and assert

```java
@Test
void shouldReturnOnlyActiveUserNames() {

    // Arrange
    UserService userService = new UserService();

    // Act
    List<User> users = userService.getActiveUsers();

    // Assert — extract specific field from objects in list
    assertThat(users)
        .extracting(User::getName)             // pull out just names
        .containsExactlyInAnyOrder("Alice", "Carol");

    // Extract multiple fields
    assertThat(users)
        .extracting("name", "age")             // extract 2 fields
        .containsExactly(
            tuple("Alice", 25),
            tuple("Carol", 30)
        );

    // Failure message if wrong:
    // expected: ["Alice", "Carol"] but was: ["Alice", "Bob"]
}
```

---

### 11. Number assertions — `isGreaterThan`, `isBetween`, `isCloseTo`

```java
@Test
void shouldApplyCorrectDiscount() {

    // Arrange
    PriceService priceService = new PriceService();

    // Act
    double discountedPrice = priceService.applyDiscount(1000.0, 10);

    // Assert
    assertThat(discountedPrice).isEqualTo(900.0);
    assertThat(discountedPrice).isGreaterThan(0);
    assertThat(discountedPrice).isLessThan(1000.0);
    assertThat(discountedPrice).isBetween(800.0, 1000.0);

    // For floating point — avoid exact equality!
    double tax = priceService.calculateTax(100.0);
    assertThat(tax).isCloseTo(18.0, within(0.01));
    // ✅ passes if value is between 17.99 and 18.01

    // Failure message if wrong:
    // expected 900.0 to be close to 18.0 within 0.01 but was 18.05
}
```

---

### 12. String assertions — `startsWith`, `endsWith`, `matches`

```java
@Test
void shouldGenerateValidEmailFormat() {

    // Arrange
    User user = new User("Alice");

    // Act
    String email = user.getEmail();

    // Assert
    assertThat(email)
        .isNotBlank()
        .isNotEmpty()
        .startsWith("alice")
        .endsWith(".com")
        .contains("@")
        .doesNotContain("  ")           // no spaces
        .matches("[a-z.]+@[a-z]+\\.com") // regex match
        .hasSize(17);

    // Failure message if wrong:
    // expected "bob@gmail.com" to start with "alice"
}
```

---

### 13. Object assertions — `hasFieldOrPropertyWithValue`

```java
@Test
void shouldCreateUserWithCorrectDetails() {

    // Arrange
    UserService userService = new UserService();

    // Act
    User user = userService.createUser("Alice", 25, "alice@gmail.com");

    // Assert — check specific fields
    assertThat(user)
        .isNotNull()
        .hasFieldOrPropertyWithValue("name",  "Alice")
        .hasFieldOrPropertyWithValue("age",   25)
        .hasFieldOrPropertyWithValue("active", true);

    // Or use extracting
    assertThat(user)
        .extracting(User::getName, User::getAge)
        .containsExactly("Alice", 25);

    // Failure message if wrong:
    // expected field "name" to be "Alice" but was "Bob"
}
```

---

### 14. `assertAll` — run ALL assertions even if one fails

```java
@Test
void shouldVerifyAllUserFields() {

    // Arrange
    User user = new User("Alice", 25, "alice@gmail.com", true);

    // Act — nothing, checking state

    // Assert — ALL run even if first one fails!
    assertAll("user validation",
        () -> assertThat(user.getName()).isEqualTo("Alice"),
        () -> assertThat(user.getAge()).isEqualTo(25),
        () -> assertThat(user.getEmail()).contains("@"),
        () -> assertThat(user.isActive()).isTrue()
    );

    // Without assertAll — if first fails, others don't run
    // With assertAll — ALL run, ALL failures reported together ✅

    // Failure message shows ALL failures at once:
    // Multiple failures:
    // 1) expected "Bob" but was "Alice"
    // 2) expected 30 but was 25
}
```

---

### 15. `as()` — custom failure message

```java
@Test
void shouldActivateUserAfterRegistration() {

    // Arrange
    UserService userService = new UserService();

    // Act
    User user = userService.register("alice@gmail.com");

    // Assert — with custom label for better error messages
    assertThat(user.isActive())
        .as("User should be active after registration")
        .isTrue();

    assertThat(user.getEmail())
        .as("Email should match registration input")
        .isEqualTo("alice@gmail.com");

    // Failure message:
    // [User should be active after registration]
    // expected: true but was: false
    // ← much clearer than just "expected true but was false"!
}
```

---

### Quick reference — all assertions

| Assertion | Checks | Example |
|---|---|---|
| `isEqualTo(x)` | exact value match | `assertThat(name).isEqualTo("Alice")` |
| `isNotEqualTo(x)` | values differ | `assertThat(id1).isNotEqualTo(id2)` |
| `isTrue()` | boolean true | `assertThat(result).isTrue()` |
| `isFalse()` | boolean false | `assertThat(result).isFalse()` |
| `isNull()` | value is null | `assertThat(user).isNull()` |
| `isNotNull()` | value not null | `assertThat(user).isNotNull()` |
| `isInstanceOf(X)` | type check | `assertThat(ex).isInstanceOf(RuntimeException.class)` |
| `contains(x)` | list has element | `assertThat(list).contains("ADMIN")` |
| `hasSize(n)` | collection size | `assertThat(list).hasSize(3)` |
| `isEmpty()` | empty collection | `assertThat(list).isEmpty()` |
| `isNotEmpty()` | not empty | `assertThat(list).isNotEmpty()` |
| `extracting(fn)` | pull field | `assertThat(users).extracting(User::getName)` |
| `isGreaterThan(x)` | number compare | `assertThat(price).isGreaterThan(0)` |
| `isBetween(a,b)` | range check | `assertThat(age).isBetween(18, 60)` |
| `isCloseTo(x,d)` | float check | `assertThat(tax).isCloseTo(18.0, within(0.01))` |
| `startsWith(x)` | string start | `assertThat(email).startsWith("alice")` |
| `matches(regex)` | regex match | `assertThat(email).matches("[a-z]+@[a-z]+\\.com")` |
| `assertAll(...)` | run all checks | `assertAll(() -> ..., () -> ...)` |
| `as("label")` | custom message | `assertThat(x).as("should be active").isTrue()` |

---

### Bottom line

> Assertions are the **verdict of your test** — they answer "did the code do what it should?"
>
> Prefer **AssertJ** (`assertThat(...)`) over plain JUnit assertions — it gives **better error messages**, **fluent chaining**, and **richer methods**
>
> One golden rule — **assert ONE concept per test** — when a test fails, you instantly know exactly what broke and why



---

### What is `assertArrayEquals()`?

> It checks that **two arrays are equal** — same length, same elements, same order.

```java
// Why can't we use assertEquals() for arrays?
int[] arr1 = {1, 2, 3};
int[] arr2 = {1, 2, 3};

assertEquals(arr1, arr2);      // ❌ FAILS!
// uses == → compares memory address → different objects → FAILS

assertArrayEquals(arr1, arr2); // ✅ PASSES!
// compares element by element → 1==1, 2==2, 3==3 → PASSES
```

---

### Why `assertEquals` FAILS for arrays

```java
@Test
void shouldShowWhyEqualsFailsForArrays() {

    int[] arr1 = {1, 2, 3}; // address: 0x001
    int[] arr2 = {1, 2, 3}; // address: 0x002

    // Arrays do NOT override .equals()
    // So .equals() falls back to Object.equals() → reference check
    assertEquals(arr1, arr2);
    // ❌ FAILS
    // expected: [I@1a2b3c
    // but was:  [I@4d5e6f
    // ← cryptic memory addresses — useless error message!
}
```

---

### Basic `assertArrayEquals` — primitive arrays

#### int array

```java
@Test
void shouldAssertIntArray() {

    // Arrange
    int[] expected = {1, 2, 3, 4, 5};

    // Act
    int[] actual = mathService.getFirstFiveNumbers();

    // Assert
    assertArrayEquals(expected, actual);
    // checks: length same? element by element same?
    // 1==1 ✅ 2==2 ✅ 3==3 ✅ 4==4 ✅ 5==5 ✅ → PASSES

    // Failure message if wrong:
    // array differ at element [2]:
    // expected: 3 but was: 99
    //                ↑ tells exact index that failed!
}
```

#### double array — with delta

```java
@Test
void shouldAssertDoubleArray() {

    // Arrange
    double[] expected = {1.1, 2.2, 3.3};

    // Act
    double[] actual = calculationService.getValues();

    // Assert — for doubles ALWAYS use delta!
    assertArrayEquals(expected, actual, 0.001);
    //                                  ↑
    //                              delta/tolerance
    // passes if each element is within 0.001 of expected
    // expected[0]=1.1, actual[0]=1.1001 → difference=0.0001 < 0.001 ✅

    // WHY delta for doubles?
    // Floating point math is imprecise:
    // 0.1 + 0.2 = 0.30000000000000004 (not exactly 0.3!)
    // delta handles this imprecision ✅
}
```

#### boolean array

```java
@Test
void shouldAssertBooleanArray() {

    // Arrange
    boolean[] expected = {true, false, true, true};

    // Act
    boolean[] actual = userService.getActiveFlags();
    // checks which users are active

    // Assert
    assertArrayEquals(expected, actual);
    // true==true ✅ false==false ✅ true==true ✅ true==true ✅
}
```

#### char array

```java
@Test
void shouldAssertCharArray() {

    // Arrange
    char[] expected = {'h', 'e', 'l', 'l', 'o'};

    // Act
    char[] actual = stringService.toCharArray("hello");

    // Assert
    assertArrayEquals(expected, actual);
    // 'h'=='h' ✅ 'e'=='e' ✅ 'l'=='l' ✅ ...
}
```

---

### String array

```java
@Test
void shouldAssertStringArray() {

    // Arrange
    String[] expected = {"Alice", "Bob", "Carol"};

    // Act
    String[] actual = userService.getAllUserNames();

    // Assert
    assertArrayEquals(expected, actual);
    // uses String.equals() for each element
    // "Alice".equals("Alice") ✅
    // "Bob".equals("Bob")     ✅
    // "Carol".equals("Carol") ✅

    // Failure if order is wrong:
    // String[] actual = {"Bob", "Alice", "Carol"};
    // array differ at element [0]:
    // expected: "Alice" but was: "Bob" ❌
}
```

---

### Object array — needs `equals()` override

```java
@Test
void shouldAssertObjectArray() {

    // Arrange
    // User MUST have equals() overridden!
    User[] expected = {
        new User("Alice", 25),
        new User("Bob",   30)
    };

    // Act
    User[] actual = userService.getTopUsers();

    // Assert
    assertArrayEquals(expected, actual);
    // calls equals() on each User object
    // user1.equals(user1) ✅
    // user2.equals(user2) ✅

    // ⚠️ If User does NOT override equals():
    // uses reference comparison → ❌ FAILS even with same data!
}
```

---

### 2D array — nested array comparison

```java
@Test
void shouldAssert2DArray() {

    // Arrange
    int[][] expected = {
        {1, 2, 3},
        {4, 5, 6},
        {7, 8, 9}
    };

    // Act
    int[][] actual = matrixService.createMatrix();

    // Assert
    assertArrayEquals(expected, actual);
    // checks each row array element by element
    // row[0]: 1==1 ✅ 2==2 ✅ 3==3 ✅
    // row[1]: 4==4 ✅ 5==5 ✅ 6==6 ✅
    // row[2]: 7==7 ✅ 8==8 ✅ 9==9 ✅

    // Failure message for 2D:
    // array differ at element [1][2]:
    // expected: 6 but was: 99 ← tells EXACT row and column! ✅
}
```

---

### Common failure scenarios

#### Different length

```java
@Test
void shouldFailWhenDifferentLength() {

    int[] expected = {1, 2, 3};
    int[] actual   = {1, 2};      // missing element!

    assertArrayEquals(expected, actual);
    // ❌ FAILS
    // array lengths differed:
    // expected: 3 but was: 2
}
```

#### Same elements, different order

```java
@Test
void shouldFailWhenDifferentOrder() {

    String[] expected = {"Alice", "Bob", "Carol"};
    String[] actual   = {"Bob", "Alice", "Carol"}; // order changed!

    assertArrayEquals(expected, actual);
    // ❌ FAILS — order matters!
    // array differ at element [0]:
    // expected: "Alice" but was: "Bob"

    // If order doesn't matter → use AssertJ instead:
    assertThat(actual)
        .containsExactlyInAnyOrder("Alice", "Bob", "Carol"); // ✅
}
```

#### null array

```java
@Test
void shouldHandleNullArrays() {

    // Both null → PASSES ✅
    assertArrayEquals(null, null); // ✅

    // One null → FAILS ❌
    int[] expected = {1, 2, 3};
    assertArrayEquals(expected, null);
    // ❌ FAILS
    // expected array but was: null
}
```

---

### `assertArrayEquals` vs AssertJ for arrays

```java
// JUnit assertArrayEquals:
assertArrayEquals(new int[]{1,2,3}, actual);

// AssertJ equivalent — more readable, more options:
assertThat(actual)
    .containsExactly(1, 2, 3);             // exact order
    .containsExactlyInAnyOrder(3, 1, 2);   // any order
    .contains(1, 2);                        // at least these
    .hasSize(3);                            // length check
    .startsWith(1);                         // first element
    .endsWith(3);                           // last element
    .doesNotContain(99);                    // absence check
```

---

### Summary table

| Array type | Usage | Note |
|---|---|---|
| `int[]`, `long[]`, `byte[]` | `assertArrayEquals(expected, actual)` | Direct comparison |
| `double[]`, `float[]` | `assertArrayEquals(expected, actual, delta)` | Always use delta! |
| `String[]` | `assertArrayEquals(expected, actual)` | Uses String.equals() |
| `Object[]` (custom) | `assertArrayEquals(expected, actual)` | Must override equals()! |
| `int[][]` (2D) | `assertArrayEquals(expected, actual)` | Compares row by row |
| Any order | Use AssertJ `containsExactlyInAnyOrder` | Order independent |

---

### Bottom line

> `assertArrayEquals()` compares arrays **element by element** — something `assertEquals()` cannot do for arrays since arrays don't override `.equals()`
>
> Always use **delta** for `double[]` and `float[]` arrays due to floating point imprecision
>
> For **custom object arrays** — always override `equals()` and `hashCode()` or the comparison falls back to reference equality and fails
>
> If **order doesn't matter** — use AssertJ's `containsExactlyInAnyOrder()` instead




![alt text](image.png)


`assertIterableEquals()` is specifically designed for comparing any `Iterable` collections. Let me explain it fully.

---

### What is `assertIterableEquals()`?

> It checks that **two Iterable collections** have the **same elements in the same order**, where each element is compared using `.equals()`.

```java
// Works with ANY Iterable:
List<>    → ✅
Set<>     → ✅
Queue<>   → ✅
LinkedList<> → ✅
Any custom Iterable → ✅
```

---

### Why not use `assertEquals()` for collections?

```java
@Test
void shouldShowWhyEqualsFailsForCollections() {

    List<String> list1 = List.of("Alice", "Bob");
    List<String> list2 = List.of("Alice", "Bob");

    // assertEquals on List — actually works for List
    // because List DOES override .equals()
    assertEquals(list1, list2); // ✅ works

    // BUT — comparing List vs LinkedList with assertEquals:
    List<String>       list   = List.of("Alice", "Bob");
    LinkedList<String> linked = new LinkedList<>(List.of("Alice", "Bob"));

    assertEquals(list, linked);         // ⚠️ may fail — different types!

    assertIterableEquals(list, linked); // ✅ PASSES — only content matters!
    // assertIterableEquals doesn't care about TYPE
    // only cares about ELEMENTS and ORDER
}
```

---

### Basic example — List of Strings

```java
@Test
void shouldAssertStringList() {

    // Arrange
    List<String> expected = List.of("Alice", "Bob", "Carol");

    // Act
    List<String> actual = userService.getAllUserNames();

    // Assert
    assertIterableEquals(expected, actual);
    // checks element by element:
    // "Alice".equals("Alice") ✅
    // "Bob".equals("Bob")     ✅
    // "Carol".equals("Carol") ✅

    // Failure message if wrong:
    // iterable differ at element [1]:
    // expected: "Bob" but was: "Dave"
    //                 ↑ tells exact index! ✅
}
```

---

### List of Integers

```java
@Test
void shouldAssertIntegerList() {

    // Arrange
    List<Integer> expected = List.of(1, 2, 3, 4, 5);

    // Act
    List<Integer> actual = mathService.getFirstFiveNumbers();

    // Assert
    assertIterableEquals(expected, actual);
    // 1==1 ✅ 2==2 ✅ 3==3 ✅ 4==4 ✅ 5==5 ✅
}
```

---

### List of custom Objects — needs `equals()` override

```java
// User class MUST override equals() and hashCode()
@Data  // Lombok generates equals, hashCode, toString
public class User {
    private String name;
    private int age;
    private String email;
}
```

```java
@Test
void shouldAssertUserList() {

    // Arrange
    List<User> expected = List.of(
        new User("Alice", 25, "alice@gmail.com"),
        new User("Bob",   30, "bob@gmail.com")
    );

    // Act
    List<User> actual = userService.getActiveUsers();

    // Assert
    assertIterableEquals(expected, actual);
    // calls equals() on each User:
    // user1.equals(user1) ✅
    // user2.equals(user2) ✅

    // ⚠️ Without equals() override:
    // compares references → FAILS even if same data! ❌
}
```

---

### Different Iterable types — same content

```java
@Test
void shouldAssertDifferentIterableTypes() {

    // Arrange — different types, same content
    List<String>       list       = List.of("A", "B", "C");
    LinkedList<String> linkedList = new LinkedList<>(List.of("A","B","C"));
    ArrayDeque<String> deque      = new ArrayDeque<>(List.of("A","B","C"));

    // Act
    Iterable<String> actual = userService.getNames(); // returns some Iterable

    // Assert — doesn't matter what TYPE the collection is
    assertIterableEquals(list,       actual); // ✅
    assertIterableEquals(linkedList, actual); // ✅
    assertIterableEquals(deque,      actual); // ✅

    // Only CONTENT and ORDER matter — not the collection type!
}
```

---

### Nested Iterables — List of Lists

```java
@Test
void shouldAssertNestedIterables() {

    // Arrange
    List<List<String>> expected = List.of(
        List.of("Alice", "Bob"),      // group 1
        List.of("Carol", "Dave"),     // group 2
        List.of("Eve")                // group 3
    );

    // Act
    List<List<String>> actual = groupService.getUserGroups();

    // Assert
    assertIterableEquals(expected, actual);
    // checks outer list element by element
    // for each element (which is also a List) → checks inner elements
    // group1: "Alice"=="Alice" ✅ "Bob"=="Bob"   ✅
    // group2: "Carol"=="Carol" ✅ "Dave"=="Dave" ✅
    // group3: "Eve"=="Eve"     ✅
}
```

---

### Common failure scenarios

#### Different size

```java
@Test
void shouldFailWhenDifferentSize() {

    List<String> expected = List.of("Alice", "Bob", "Carol");
    List<String> actual   = List.of("Alice", "Bob"); // missing Carol!

    assertIterableEquals(expected, actual);
    // ❌ FAILS
    // iterable lengths differ:
    // expected: 3 but was: 2
}
```

#### Same elements, wrong order

```java
@Test
void shouldFailWhenWrongOrder() {

    List<String> expected = List.of("Alice", "Bob", "Carol");
    List<String> actual   = List.of("Bob", "Alice", "Carol"); // order changed!

    assertIterableEquals(expected, actual);
    // ❌ FAILS — order matters!
    // iterable differ at element [0]:
    // expected: "Alice" but was: "Bob"

    // If order doesn't matter — use AssertJ:
    assertThat(actual)
        .containsExactlyInAnyOrder("Alice", "Bob", "Carol"); // ✅
}
```

#### null elements inside list

```java
@Test
void shouldHandleNullElements() {

    List<String> expected = List.of("Alice", null, "Carol");
    List<String> actual   = Arrays.asList("Alice", null, "Carol");

    assertIterableEquals(expected, actual); // ✅ PASSES
    // null == null → treated as equal ✅
}
```

#### Both null iterables

```java
@Test
void shouldHandleNullIterables() {

    // Both null → PASSES ✅
    assertIterableEquals(null, null); // ✅

    // One null → FAILS ❌
    List<String> expected = List.of("Alice");
    assertIterableEquals(expected, null);
    // ❌ FAILS
    // expected iterable but was: null
}
```

---

### `assertIterableEquals` vs `assertArrayEquals` vs AssertJ

```java
List<String> expected = List.of("Alice", "Bob");
List<String> actual   = userService.getNames();

// JUnit — assertArrayEquals (arrays only)
assertArrayEquals(
    expected.toArray(),
    actual.toArray()
);  // works but awkward for collections

// JUnit — assertIterableEquals (any Iterable) ✅
assertIterableEquals(expected, actual);

// AssertJ — most readable, most options ✅✅
assertThat(actual)
    .containsExactly("Alice", "Bob")          // exact order
    .containsExactlyInAnyOrder("Bob", "Alice") // any order
    .contains("Alice")                         // partial match
    .hasSize(2)                                // size check
    .doesNotContain("Carol");                  // absence check
```

---

### When to use what

```
assertArrayEquals()     → when you have primitive or object ARRAYS
assertIterableEquals()  → when you have List, Set, Queue, any Iterable
AssertJ containsExactly → when you want more readable assertions
                          or order doesn't matter
```

---

### Summary table

| Scenario | Use |
|---|---|
| Same elements, same order | `assertIterableEquals(expected, actual)` |
| Same elements, any order | AssertJ `containsExactlyInAnyOrder(...)` |
| Different Iterable types (List vs LinkedList) | `assertIterableEquals` — type ignored ✅ |
| Custom objects | Must override `equals()` and `hashCode()` |
| Nested collections | `assertIterableEquals` handles recursively |
| Partial match | AssertJ `contains(...)` |
| Both null | `assertIterableEquals(null, null)` → passes |

---

### Bottom line

> `assertIterableEquals()` compares **any two Iterables element by element** in order — it doesn't care about the collection **type**, only the **content**
>
> For **custom objects** — always override `equals()` and `hashCode()`, otherwise comparison falls back to reference equality and fails
>
> If **order doesn't matter** or you need **partial matching** — use AssertJ's `containsExactlyInAnyOrder()` or `contains()` instead




`assertLinesMatch()` is a unique and powerful assertion specifically for comparing **lists of strings line by line**. Let me explain it fully.

---

### What is `assertLinesMatch()`?

> It compares **two lists of Strings** line by line — but with a superpower: it supports **regular expressions** and **fast-forward markers** that no other assertion has.

```java
// Basic signature:
assertLinesMatch(List<String> expectedLines, List<String> actualLines);
```

---

### Why not just use `assertIterableEquals()` for Strings?

```java
// assertIterableEquals — EXACT match only
assertIterableEquals(
    List.of("Hello", "World"),
    List.of("Hello", "World")
); // ✅ works but...

// What if actual has a TIMESTAMP or DYNAMIC value?
// actual = ["Hello", "World - generated at 2026-05-29 10:30:45"]
// assertIterableEquals FAILS — can't handle dynamic content ❌

// assertLinesMatch — supports REGEX!
assertLinesMatch(
    List.of("Hello", "World - generated at \\d{4}-\\d{2}-\\d{2}.*"),
    actual
); // ✅ PASSES — regex matches dynamic timestamp!
```

---

### 3 ways `assertLinesMatch` matches each line

```
1. EXACT match      → "hello" matches "hello"
2. REGEX match      → "\\d+" matches "12345"
3. FAST-FORWARD     → ">> N >>" skips N lines
```

---

### Way 1 — Exact String match (basic usage)

```java
@Test
void shouldMatchExactLines() {

    // Arrange
    List<String> expected = List.of(
        "Starting application",
        "Loading configuration",
        "Server started successfully"
    );

    // Act
    List<String> actual = appService.getStartupLogs();

    // Assert
    assertLinesMatch(expected, actual);
    // Line 0: "Starting application"    == "Starting application"    ✅
    // Line 1: "Loading configuration"   == "Loading configuration"   ✅
    // Line 2: "Server started..."       == "Server started..."       ✅

    // Failure message:
    // Line 1 ==> expected: "Loading configuration"
    //            but was:  "Loading config"
}
```

---

### Way 2 — Regex match (the superpower!)

```java
@Test
void shouldMatchLinesWithRegex() {

    // Arrange — expected uses REGEX patterns
    List<String> expected = List.of(
        "User created: \\w+",           // any word character
        "Timestamp: \\d{4}-\\d{2}-\\d{2}", // date pattern
        "Order ID: ORD-\\d+",           // ORD- followed by numbers
        "Amount: \\$[0-9]+\\.[0-9]{2}", // currency format
        "Status: (SUCCESS|FAILED)"      // either value
    );

    // Act — actual has dynamic/generated values
    List<String> actual = List.of(
        "User created: Alice",           // ✅ matches \w+
        "Timestamp: 2026-05-29",         // ✅ matches date pattern
        "Order ID: ORD-98765",           // ✅ matches ORD-\d+
        "Amount: $1500.00",              // ✅ matches currency
        "Status: SUCCESS"                // ✅ matches (SUCCESS|FAILED)
    );

    // Assert
    assertLinesMatch(expected, actual); // ✅ ALL PASS!

    // How JUnit decides if regex or exact:
    // → tries exact match first
    // → if exact fails → tries as regex
    // → if regex also fails → line fails ❌
}
```

---

### Way 3 — Fast-forward marker `>> N >>`

Fast-forward **skips N lines** in actual — useful when you don't care about some lines:

```java
@Test
void shouldSkipLinesWithFastForward() {

    // Arrange
    List<String> expected = List.of(
        "Application starting",
        ">> 3 >>",                  // ← skip next 3 lines in actual
        "Server ready on port 8080"
    );

    // Act — actual has extra lines we don't care about
    List<String> actual = List.of(
        "Application starting",     // ✅ matches line 0
        "Loading beans...",         // ⏭️ skipped by >> 3 >>
        "Scanning components...",   // ⏭️ skipped
        "Initializing cache...",    // ⏭️ skipped
        "Server ready on port 8080" // ✅ matches after skip
    );

    // Assert
    assertLinesMatch(expected, actual); // ✅ PASSES!
}
```

---

### Fast-forward WITHOUT a number `>> >>`

Skips lines until the **next expected line is found**:

```java
@Test
void shouldSkipUntilNextMatch() {

    // Arrange
    List<String> expected = List.of(
        "App started",
        ">> >>",                   // ← skip ANY number of lines
        "Shutdown complete"        // until this line is found
    );

    // Act — unknown number of lines in between
    List<String> actual = List.of(
        "App started",             // ✅ matches
        "Processing request 1",   // ⏭️ skipped
        "Processing request 2",   // ⏭️ skipped
        "Processing request 3",   // ⏭️ skipped
        "Closing connections",    // ⏭️ skipped
        "Shutdown complete"       // ✅ found and matched!
    );

    // Assert
    assertLinesMatch(expected, actual); // ✅ PASSES!

    // Perfect for log files where middle content varies!
}
```

---

### Real world — log file validation

```java
@Test
void shouldValidateApplicationLogs() {

    // Arrange — mix of exact, regex, and fast-forward
    List<String> expectedLogs = List.of(
        "INFO  - Application starting",                    // exact
        "INFO  - Loading properties from \\S+",           // regex - any filename
        ">> >>",                                           // skip Spring boot banner
        "INFO  - Started .+ in \\d+\\.\\d+ seconds",      // regex - startup time
        ">> 2 >>",                                         // skip 2 more lines
        "INFO  - Server listening on port \\d+",          // regex - any port
        "DEBUG - Database connected: \\w+"                // regex - any db name
    );

    // Act
    List<String> actualLogs = logService.getApplicationLogs();
    // actual might be:
    // "INFO  - Application starting"
    // "INFO  - Loading properties from application.properties"
    // "  .   ____          _            __ _ _"    ← Spring banner
    // " /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \"  ← Spring banner
    // "INFO  - Started MyApp in 3.245 seconds"
    // "INFO  - Tomcat initialized"
    // "INFO  - Tomcat started"
    // "INFO  - Server listening on port 8080"
    // "DEBUG - Database connected: mydb"

    // Assert
    assertLinesMatch(expectedLogs, actualLogs); // ✅ PASSES!
}
```

---

### Email content validation

```java
@Test
void shouldValidateEmailContent() {

    // Arrange
    List<String> expectedEmailLines = List.of(
        "Dear \\w+,",                          // any name
        "",                                     // blank line
        "Your order ORD-\\d+ has been placed.", // any order ID
        "Total amount: \\$[0-9,]+\\.[0-9]{2}", // any amount
        ">> >>",                                // skip address lines
        "Thank you for shopping with us!",      // exact
        "",                                     // blank line
        "Regards,",                             // exact
        "The .+ Team"                           // any team name
    );

    // Act
    List<String> actualLines = emailService
        .generateOrderEmail(order)
        .getLines();

    // Assert
    assertLinesMatch(expectedEmailLines, actualLines); // ✅
}
```

---

### Common failure scenarios

#### Line content mismatch

```java
@Test
void shouldFailWhenLineMismatch() {

    List<String> expected = List.of("Hello", "World");
    List<String> actual   = List.of("Hello", "Java");

    assertLinesMatch(expected, actual);
    // ❌ FAILS
    // Line 1 ==> expected: "World"
    //            but was:  "Java"
    // also tried to match as regular expression
    // but "World" regex didn't match "Java"
}
```

#### Different number of lines

```java
@Test
void shouldFailWhenDifferentLineCount() {

    List<String> expected = List.of("Line1", "Line2", "Line3");
    List<String> actual   = List.of("Line1", "Line2");

    assertLinesMatch(expected, actual);
    // ❌ FAILS
    // expected 3 lines but actual has only 2 lines
}
```

#### Invalid fast-forward — skips too many

```java
@Test
void shouldFailWhenFastForwardTooMany() {

    List<String> expected = List.of(
        "Start",
        ">> 10 >>",   // skip 10 lines
        "End"
    );

    List<String> actual = List.of(
        "Start",
        "Middle",     // only 1 line — can't skip 10!
        "End"
    );

    assertLinesMatch(expected, actual);
    // ❌ FAILS
    // fast-forward >> 10 >> — not enough lines remaining
}
```

---

### `assertLinesMatch` vs other assertions

```java
List<String> actual = logService.getLogs();

// assertIterableEquals — EXACT match only, no regex
assertIterableEquals(expected, actual);
// ❌ fails if any line has dynamic content

// assertLinesMatch — exact + regex + fast-forward
assertLinesMatch(expected, actual);
// ✅ handles dynamic content with regex

// AssertJ — most flexible
assertThat(actual)
    .hasSize(5)
    .contains("Server started")
    .anyMatch(line -> line.matches(".*\\d{4}-\\d{2}-\\d{2}.*"));
// ✅ most options but no fast-forward
```

---

### Quick reference — special markers

```
">> N >>"   → skip exactly N lines
">> >>"     → skip until next expected line found
"\\d+"      → regex: one or more digits
"\\w+"      → regex: one or more word characters
"\\S+"      → regex: one or more non-whitespace
".+"        → regex: one or more any character
"(A|B)"     → regex: either A or B
```

---

### Summary table

| Feature | `assertLinesMatch` | `assertIterableEquals` | AssertJ |
|---|---|---|---|
| Exact match | ✅ | ✅ | ✅ |
| Regex match | ✅ | ❌ | ✅ (anyMatch) |
| Skip lines | ✅ `>> N >>` | ❌ | ❌ |
| Dynamic content | ✅ | ❌ | ✅ |
| Order matters | ✅ | ✅ | Configurable |
| Best for | Log files, text output | Simple lists | Flexible assertions |

---

### Bottom line

> `assertLinesMatch()` is purpose-built for comparing **lists of strings** where content may be **dynamic or partially variable**
>
> Its 3 powers — **exact match**, **regex match**, and **fast-forward skipping** — make it perfect for validating **log files, email content, generated reports**, or any multi-line text output
>
> Use it when `assertIterableEquals()` fails because of **timestamps, IDs, or other generated values** that change every run


---

### Why `\\` (double backslash) in Java?

Before diving in — this is a common confusion:

```java
// In REGEX:    \d  means digit
// In Java STRING: \ needs to be escaped as \\
// So in Java code: "\\d" = regex \d

String regex = "\\d+";  // Java string
// Java sees:  \d+      // actual regex pattern
// means:      one or more digits
```

```
Regex world  →  Java String world
    \d       →     \\d
    \w       →     \\w
    \s       →     \\s
    \.       →     \\.
```

---

### Pattern 1 — `\\d+` — Digits only

```
\d  = any single digit (0-9)
+   = one or more times
\\d+ in Java = one or more digits
```

```java
@Test
void shouldUnderstandDigitPattern() {

    // \\d+ matches:
    String p = "\\d+";

    assertTrue("123".matches(p));      // ✅ three digits
    assertTrue("42".matches(p));       // ✅ two digits
    assertTrue("0".matches(p));        // ✅ single digit
    assertTrue("9999999".matches(p));  // ✅ many digits

    assertFalse("abc".matches(p));     // ❌ letters — not digits
    assertFalse("12.3".matches(p));    // ❌ has decimal point
    assertFalse("".matches(p));        // ❌ empty — needs at least 1

    // Variants:
    // \\d    = exactly ONE digit
    // \\d+   = ONE or more digits
    // \\d*   = ZERO or more digits
    // \\d{4} = EXACTLY 4 digits
    // \\d{2,4} = 2 to 4 digits
}
```

Real use in `assertLinesMatch`:

```java
@Test
void shouldMatchDigitPatternInLogs() {

    List<String> expected = List.of(
        "Order ID: \\d+",           // any order number
        "Items count: \\d+",        // any count
        "Year: \\d{4}",             // exactly 4 digits
        "Port: \\d{4,5}",           // 4 or 5 digits (1000-99999)
        "Date: \\d{2}-\\d{2}-\\d{4}" // DD-MM-YYYY format
    );

    List<String> actual = List.of(
        "Order ID: 98765",          // ✅ matches \\d+
        "Items count: 3",           // ✅ matches \\d+
        "Year: 2026",               // ✅ matches \\d{4}
        "Port: 8080",               // ✅ matches \\d{4,5}
        "Date: 29-05-2026"          // ✅ matches date pattern
    );

    assertLinesMatch(expected, actual); // ✅ ALL PASS
}
```

---

### Pattern 2 — `[A-Za-z]+` — Letters only

```
[A-Za-z]  = any single letter (uppercase OR lowercase)
+         = one or more times
Matches ONLY letters — no digits, no spaces
```

```java
@Test
void shouldUnderstandLetterPattern() {

    String p = "[A-Za-z]+";

    assertTrue("Hello".matches(p));    // ✅ all letters
    assertTrue("World".matches(p));    // ✅ all letters
    assertTrue("ABC".matches(p));      // ✅ uppercase
    assertTrue("xyz".matches(p));      // ✅ lowercase
    assertTrue("JavaTest".matches(p)); // ✅ mixed case

    assertFalse("Hello123".matches(p)); // ❌ has digits
    assertFalse("Hello World".matches(p)); // ❌ has space
    assertFalse("test_value".matches(p));  // ❌ has underscore
    assertFalse("".matches(p));            // ❌ empty

    // Variants:
    // [A-Z]+     = ONLY uppercase letters
    // [a-z]+     = ONLY lowercase letters
    // [A-Za-z]+  = any letter upper or lower
    // [A-Za-z]   = exactly ONE letter
    // [A-Za-z]{3} = exactly 3 letters
}
```

Real use in `assertLinesMatch`:

```java
@Test
void shouldMatchLetterPatternInOutput() {

    List<String> expected = List.of(
        "Status: [A-Z]+",           // uppercase status word
        "Username: [A-Za-z]+",      // any name (letters only)
        "Country code: [A-Z]{2}",   // exactly 2 uppercase letters
        "Month: [A-Za-z]{3,9}"      // 3-9 letters (Jan to September)
    );

    List<String> actual = List.of(
        "Status: ACTIVE",           // ✅ uppercase letters
        "Username: Alice",          // ✅ mixed case letters
        "Country code: IN",         // ✅ exactly 2 uppercase
        "Month: January"            // ✅ 7 letters
    );

    assertLinesMatch(expected, actual); // ✅ ALL PASS
}
```

---

### Pattern 3 — `\\w+` — Alphanumeric + underscore

```
\w  = [A-Za-z0-9_]  (letters + digits + underscore)
+   = one or more times

\\w+ is MORE POWERFUL than [A-Za-z]+
because it also includes digits and underscore
```

```java
@Test
void shouldUnderstandAlphanumericPattern() {

    String p = "\\w+";

    assertTrue("Hello".matches(p));      // ✅ letters
    assertTrue("Hello123".matches(p));   // ✅ letters + digits
    assertTrue("user_name".matches(p));  // ✅ with underscore
    assertTrue("test_1".matches(p));     // ✅ mixed
    assertTrue("ABC123_xyz".matches(p)); // ✅ all types
    assertTrue("123".matches(p));        // ✅ only digits

    assertFalse("Hello World".matches(p)); // ❌ space not allowed
    assertFalse("user-name".matches(p));   // ❌ hyphen not allowed
    assertFalse("test.value".matches(p));  // ❌ dot not allowed
    assertFalse("".matches(p));            // ❌ empty

    // Key difference:
    // [A-Za-z]+ → letters ONLY
    // \\w+       → letters + digits + underscore
}
```

Real use in `assertLinesMatch`:

```java
@Test
void shouldMatchAlphanumericPatternInLogs() {

    List<String> expected = List.of(
        "Username: \\w+",           // Alice, user_123, john_doe
        "Token: \\w+",              // abc123, xyz_789
        "Variable: \\w+",           // myVar, my_var_1
        "Transaction: TXN_\\w+"     // TXN_abc123, TXN_xyz
    );

    List<String> actual = List.of(
        "Username: john_doe",       // ✅ has underscore — matches \\w+
        "Token: abc123xyz",         // ✅ alphanumeric
        "Variable: my_var_1",       // ✅ underscore + digits
        "Transaction: TXN_98xyz"    // ✅ TXN_ + alphanumeric
    );

    assertLinesMatch(expected, actual); // ✅ ALL PASS
}
```

---

### Pattern 4 — `.*` — Anything (whole line)

```
.   = any single character (except newline)
*   = zero or more times
.*  = matches EVERYTHING — even empty string!
```

```java
@Test
void shouldUnderstandAnyPattern() {

    String p = ".*";

    assertTrue("Hello".matches(p));          // ✅
    assertTrue("Hello World 123!".matches(p));// ✅
    assertTrue("".matches(p));               // ✅ even empty!
    assertTrue("   ".matches(p));            // ✅ spaces
    assertTrue("!@#$%^&*".matches(p));       // ✅ special chars
    assertTrue("anything goes here".matches(p)); // ✅

    // .* matches EVERYTHING — use when you don't care
    // about that part of the line at all
}
```

Real use in `assertLinesMatch`:

```java
@Test
void shouldMatchAnyPatternInLogs() {

    List<String> expected = List.of(
        "INFO.*",                    // starts with INFO, anything after
        ".*Exception.*",             // contains Exception anywhere
        "User .* logged in",         // User + anything + logged in
        ".*",                        // match ANY line completely
        "Started .* in \\d+.* seconds" // flexible startup message
    );

    List<String> actual = List.of(
        "INFO - Server started on 2026-05-29", // ✅ starts with INFO
        "java.lang.NullPointerException: ...", // ✅ contains Exception
        "User Alice Smith logged in",          // ✅ middle part flexible
        "literally anything here !@#",         // ✅ .* matches all
        "Started MyApp in 3.245 seconds"       // ✅ flexible middle
    );

    assertLinesMatch(expected, actual); // ✅ ALL PASS
}
```

---

### Combining patterns — real power

```java
@Test
void shouldCombineMultiplePatterns() {

    List<String> expected = List.of(

        // Timestamp: 2026-05-29 10:30:45
        "Timestamp: \\d{4}-\\d{2}-\\d{2} \\d{2}:\\d{2}:\\d{2}",

        // Email: any@any.com
        "[A-Za-z\\d._%+-]+@[A-Za-z\\d.-]+\\.[A-Za-z]{2,}",

        // Log level + message
        "(INFO|DEBUG|WARN|ERROR) - .*",

        // Version: v1.2.3
        "Version: v\\d+\\.\\d+\\.\\d+",

        // UUID: 550e8400-e29b-41d4-a716-446655440000
        "[\\w]{8}-[\\w]{4}-[\\w]{4}-[\\w]{4}-[\\w]{12}"
    );

    List<String> actual = List.of(
        "Timestamp: 2026-05-29 10:30:45",           // ✅
        "alice.smith@example.com",                   // ✅
        "INFO - Application started successfully",   // ✅
        "Version: v2.5.1",                           // ✅
        "550e8400-e29b-41d4-a716-446655440000"       // ✅
    );

    assertLinesMatch(expected, actual); // ✅ ALL PASS!
}
```

---

### All patterns — quick reference

| Pattern | Matches | Example |
|---|---|---|
| `\\d` | single digit | `5` |
| `\\d+` | one or more digits | `123`, `42` |
| `\\d{4}` | exactly 4 digits | `2026` |
| `\\d{2,4}` | 2 to 4 digits | `26`, `202` |
| `[A-Z]+` | uppercase letters | `ACTIVE` |
| `[a-z]+` | lowercase letters | `hello` |
| `[A-Za-z]+` | any letters | `Hello` |
| `[A-Za-z]{2}` | exactly 2 letters | `IN`, `US` |
| `\\w` | single alphanumeric/underscore | `a`, `1`, `_` |
| `\\w+` | one or more alphanumeric/underscore | `user_123` |
| `\\s` | single whitespace | ` ` |
| `\\s+` | one or more whitespace | `   ` |
| `.` | any single character | anything |
| `.*` | any characters (whole line) | `anything!` |
| `.+` | any characters (at least 1) | `a` |
| `(A\|B)` | either A or B | `SUCCESS` or `FAILED` |
| `\\.` | literal dot | `.` |
| `\\$` | literal dollar sign | `$` |

---

### Bottom line

> These 4 regex patterns are the **building blocks** for `assertLinesMatch()`:
>
> `\\d+` → for **numbers** (IDs, counts, ports, dates)
> `[A-Za-z]+` → for **pure text** (names, words, codes)
> `\\w+` → for **identifiers** (usernames, tokens, variables with underscore)
> `.*` → for **anything** (when you don't care about that part)
>
> Combine them to build powerful patterns that handle **dynamic content** in your test output — timestamps, generated IDs, variable messages — without hardcoding values that change every run






Let me explain all of these assertions with clear examples, each with its own test method!

---

### 1. `assertSame()` — checks SAME reference (==)

```java
@Test
void shouldAssertSameReference() {

    // Arrange
    User user1 = new User("Alice", 25);
    User user2 = user1;        // same reference
    User user3 = new User("Alice", 25); // different object, same data

    // Act + Assert
    assertSame(user1, user2);    // ✅ PASSES — same memory address
    assertSame(user1, user3);    // ❌ FAILS  — different objects!

    // Internally uses ==
    // user1 == user2 → true  ✅ (same reference)
    // user1 == user3 → false ❌ (different objects)

    // Failure message:
    // expected: same instance but was not
    // expected: User@1a2b3c
    //  but was: User@4d5e6f
}
```

Real use case — **singleton or cached object**:

```java
@Test
void shouldReturnSameSingletonInstance() {

    // Arrange
    DatabaseConnection conn1 = DatabaseConnection.getInstance();
    DatabaseConnection conn2 = DatabaseConnection.getInstance();

    // Assert — must be SAME object (singleton pattern)
    assertSame(conn1, conn2); // ✅ singleton returns same instance
}
```

---

### 2. `assertNotSame()` — checks DIFFERENT reference

```java
@Test
void shouldAssertNotSameReference() {

    // Arrange
    User user1 = new User("Alice", 25);
    User user2 = new User("Alice", 25); // different object

    // Assert
    assertNotSame(user1, user2); // ✅ PASSES — different objects

    // Failure message if same:
    // expected: not same instance but was same
}
```

Real use case — **factory creates new objects each time**:

```java
@Test
void shouldCreateNewOrderEachTime() {

    // Arrange
    OrderFactory factory = new OrderFactory();

    // Act
    Order order1 = factory.createOrder();
    Order order2 = factory.createOrder();

    // Assert — factory must return NEW objects, not cached
    assertNotSame(order1, order2); // ✅ different instances
}
```

---

### 3. `assertNull()` — checks value IS null

```java
@Test
void shouldReturnNullWhenUserNotFound() {

    // Arrange
    UserRepository repo = new UserRepository();

    // Act
    User user = repo.findByEmail("unknown@gmail.com");

    // Assert
    assertNull(user); // ✅ PASSES — no user found → null returned

    // With custom message:
    assertNull(user, "User should not exist for unknown email");

    // Failure message if NOT null:
    // expected: <null> but was: User{name='Alice'}
}
```

---

### 4. `assertNotNull()` — checks value is NOT null

```java
@Test
void shouldReturnUserWhenFound() {

    // Arrange
    UserRepository repo = new UserRepository();
    repo.save(new User("Alice", 25));

    // Act
    User user = repo.findByEmail("alice@gmail.com");

    // Assert
    assertNotNull(user); // ✅ PASSES — user found → not null

    // With custom message:
    assertNotNull(user, "User must exist for valid email");

    // Failure message if null:
    // expected: not <null>
}
```

Real use case — verify object was created:

```java
@Test
void shouldCreateUserSuccessfully() {

    // Arrange
    UserService service = new UserService();

    // Act
    User created = service.createUser("Alice", 25);

    // Assert — created object must not be null
    assertNotNull(created);
    assertNotNull(created.getId());     // ID generated
    assertNotNull(created.getCreatedAt()); // timestamp set
}
```

---

### 5. `fail()` — force a test to FAIL immediately

```java
@Test
void shouldFailExplicitly() {

    // Arrange
    PaymentService service = new PaymentService();

    try {
        // Act
        service.processPayment(-1000.0); // negative amount

        // If we reach here — exception was NOT thrown
        // That's a problem — so we force fail!
        fail("Expected IllegalArgumentException but none was thrown!");
        // ❌ test fails with your custom message

    } catch (IllegalArgumentException e) {
        // ✅ Expected — exception was thrown correctly
        assertThat(e.getMessage()).contains("negative");
    }
}
```

Another use case — **incomplete test / TODO marker**:

```java
@Test
void shouldImplementPaymentRefund() {

    // Mark test as explicitly failing until feature is implemented
    fail("Test not implemented yet — waiting for refund feature");
    // ❌ Always fails — reminds team this needs implementation
}
```

Another use case — **unreachable code**:

```java
@Test
void shouldNeverReachCertainCode() {

    List<String> roles = userService.getRoles("UNKNOWN_USER");

    for (String role : roles) {
        if (role.equals("SUPER_ADMIN")) {
            fail("Unknown user should NEVER have SUPER_ADMIN role!");
        }
    }
}
```

---

### 6. `assertInstanceOf()` — checks object TYPE

```java
@Test
void shouldReturnCorrectInstanceType() {

    // Arrange
    PaymentService service = new PaymentService();

    // Act
    Payment payment = service.createPayment("UPI");

    // Assert
    assertInstanceOf(UpiPayment.class, payment);
    // ✅ PASSES if payment is instance of UpiPayment

    // Also works with parent class / interface
    assertInstanceOf(Payment.class, payment);     // ✅ parent class
    assertInstanceOf(Serializable.class, payment); // ✅ interface

    // Failure message:
    // expected instance of: UpiPayment
    //              but was: CreditCardPayment
}
```

Real use case — **polymorphism check**:

```java
@Test
void shouldCreateCorrectPaymentType() {

    // Assert different payment types
    assertInstanceOf(UpiPayment.class,
        paymentFactory.create("UPI"));        // ✅

    assertInstanceOf(CreditCardPayment.class,
        paymentFactory.create("CREDIT_CARD")); // ✅

    assertInstanceOf(NetBankingPayment.class,
        paymentFactory.create("NET_BANKING")); // ✅
}
```

With returned typed object — JUnit 5.8+:

```java
@Test
void shouldReturnTypedInstance() {

    // assertInstanceOf returns the typed object!
    Animal animal = new Dog("Rex");

    Dog dog = assertInstanceOf(Dog.class, animal);
    // ↑ returns Dog type — no casting needed!

    // Now use dog directly without cast:
    assertThat(dog.bark()).isEqualTo("Woof!");
    assertThat(dog.getName()).isEqualTo("Rex");
}
```

---

### 7. `assertThrows()` — checks exception IS thrown

```java
@Test
void shouldThrowExceptionForInvalidAge() {

    // Arrange
    UserService service = new UserService();

    // Act + Assert
    IllegalArgumentException exception =
        assertThrows(IllegalArgumentException.class,
            () -> service.createUser("Alice", -5)
            //                               ↑ invalid age
        );

    // assertThrows RETURNS the exception — check message too!
    assertThat(exception.getMessage())
        .isEqualTo("Age cannot be negative");

    // Failure if NO exception thrown:
    // expected IllegalArgumentException to be thrown
    // but nothing was thrown

    // Failure if DIFFERENT exception thrown:
    // expected IllegalArgumentException
    //  but was NullPointerException
}
```

Real use case:

```java
@Test
void shouldThrowWhenUserNotFound() {

    // Arrange
    when(userRepository.findById(999L))
        .thenReturn(Optional.empty());

    // Act + Assert
    UserNotFoundException ex =
        assertThrows(UserNotFoundException.class,
            () -> userService.findById(999L)
        );

    // Verify exception details
    assertThat(ex.getMessage()).contains("999");
    assertThat(ex.getErrorCode()).isEqualTo("USER_404");
}
```

It checks the exception thrown by the lambda is the type or subtype we mentioned on 1st parameter.

---

### What is `assertThrows()`?

> It verifies that a piece of code **throws a specific exception**. If no exception is thrown — or a **different** exception is thrown — the test **FAILS**.

```java
// Basic signature:
assertThrows(
    ExceptionClass.class,  // expected exception type
    () -> { /* code that should throw */ }
);
```

---

### Why do we need it?

```java
// ❌ OLD way — messy try/catch in tests
@Test
void oldWayToTestException() {
    try {
        userService.findById(-1L);
        fail("Should have thrown exception!"); // easy to forget this!
    } catch (IllegalArgumentException e) {
        assertEquals("Invalid ID", e.getMessage());
    }
}

// ✅ NEW way — assertThrows is clean and readable
@Test
void newWayToTestException() {
    assertThrows(IllegalArgumentException.class,
        () -> userService.findById(-1L)
    );
}
```

---

### Basic example — simplest form

```java
@Test
void shouldThrowWhenAgeIsNegative() {

    // Arrange
    UserService service = new UserService();

    // Act + Assert
    assertThrows(
        IllegalArgumentException.class,   // expected exception
        () -> service.createUser("Alice", -5) // code that throws
    );

    // Test PASSES  ✅ if IllegalArgumentException is thrown
    // Test FAILS   ❌ if NO exception is thrown
    // Test FAILS   ❌ if DIFFERENT exception is thrown
}
```

---

### `assertThrows` RETURNS the exception — use it!

This is the most important feature most developers miss:

```java
@Test
void shouldCaptureAndVerifyException() {

    // Arrange
    UserService service = new UserService();

    // Act — assertThrows RETURNS the exception!
    IllegalArgumentException exception =
        assertThrows(
            IllegalArgumentException.class,
            () -> service.createUser("Alice", -5)
        );

    // Now verify EVERYTHING about the exception:
    assertThat(exception.getMessage())
        .isEqualTo("Age cannot be negative");

    assertThat(exception.getMessage())
        .contains("negative");

    assertThat(exception.getMessage())
        .startsWith("Age");
}
```

---

### 3 possible outcomes

```java
@Test
void possibleOutcomes() {

    // OUTCOME 1: Correct exception thrown → TEST PASSES ✅
    assertThrows(IllegalArgumentException.class,
        () -> { throw new IllegalArgumentException("bad input"); }
    ); // ✅ PASSES


    // OUTCOME 2: NO exception thrown → TEST FAILS ❌
    assertThrows(IllegalArgumentException.class,
        () -> { int x = 1 + 1; } // no exception!
    );
    // ❌ FAILS
    // Expected IllegalArgumentException to be thrown
    // but nothing was thrown


    // OUTCOME 3: DIFFERENT exception thrown → TEST FAILS ❌
    assertThrows(IllegalArgumentException.class,
        () -> { throw new NullPointerException("null!"); }
    );
    // ❌ FAILS
    // Expected IllegalArgumentException
    // but NullPointerException was thrown
}
```

---

### Real world examples

#### Example 1 — Invalid user input

```java
// Production code
@Service
public class UserService {

    public User createUser(String name, int age) {
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException("Name cannot be blank");
        }
        if (age < 0) {
            throw new IllegalArgumentException("Age cannot be negative");
        }
        if (age > 150) {
            throw new IllegalArgumentException("Age is unrealistic");
        }
        return new User(name, age);
    }
}
```

```java
// Test class — separate test for each validation
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @InjectMocks
    UserService userService;

    @Test
    void shouldThrowWhenNameIsNull() {

        IllegalArgumentException ex =
            assertThrows(IllegalArgumentException.class,
                () -> userService.createUser(null, 25)
            );

        assertThat(ex.getMessage())
            .isEqualTo("Name cannot be blank");
    }

    @Test
    void shouldThrowWhenNameIsBlank() {

        IllegalArgumentException ex =
            assertThrows(IllegalArgumentException.class,
                () -> userService.createUser("   ", 25)
            );

        assertThat(ex.getMessage())
            .isEqualTo("Name cannot be blank");
    }

    @Test
    void shouldThrowWhenAgeIsNegative() {

        IllegalArgumentException ex =
            assertThrows(IllegalArgumentException.class,
                () -> userService.createUser("Alice", -1)
            );

        assertThat(ex.getMessage())
            .isEqualTo("Age cannot be negative");
    }

    @Test
    void shouldThrowWhenAgeIsUnrealistic() {

        IllegalArgumentException ex =
            assertThrows(IllegalArgumentException.class,
                () -> userService.createUser("Alice", 200)
            );

        assertThat(ex.getMessage())
            .isEqualTo("Age is unrealistic");
    }

    @Test
    void shouldNotThrowWhenInputIsValid() {

        // Valid input — should NOT throw
        assertDoesNotThrow(
            () -> userService.createUser("Alice", 25)
        );
    }
}
```

---

#### Example 2 — Custom exception with fields

```java
// Custom exception
public class UserNotFoundException extends RuntimeException {

    private final Long userId;
    private final String errorCode;

    public UserNotFoundException(Long userId) {
        super("User not found with id: " + userId);
        this.userId    = userId;
        this.errorCode = "USER_404";
    }

    public Long getUserId()     { return userId; }
    public String getErrorCode(){ return errorCode; }
}
```

```java
// Production service
@Service
public class UserService {

    @Autowired
    UserRepository userRepository;

    public User findById(Long id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
    }
}
```

```java
// Test — verify ALL exception fields
@Test
void shouldThrowUserNotFoundWithCorrectDetails() {

    // Arrange
    when(userRepository.findById(999L))
        .thenReturn(Optional.empty());

    // Act — capture the exception
    UserNotFoundException ex =
        assertThrows(UserNotFoundException.class,
            () -> userService.findById(999L)
        );

    // Assert ALL exception details
    assertThat(ex.getMessage())
        .isEqualTo("User not found with id: 999");

    assertThat(ex.getUserId())
        .isEqualTo(999L);

    assertThat(ex.getErrorCode())
        .isEqualTo("USER_404");

    // Also verify it's a RuntimeException (parent)
    assertInstanceOf(RuntimeException.class, ex);
}
```

---

#### Example 3 — Exception with cause (chained exceptions)

```java
// Production code
public class OrderService {

    public void placeOrder(OrderRequest request) {
        try {
            paymentGateway.charge(request.getAmount());
        } catch (PaymentException e) {
            // wrap in domain exception
            throw new OrderFailedException(
                "Order failed due to payment error", e
            );
        }
    }
}
```

```java
// Test — verify exception AND its cause
@Test
void shouldThrowOrderFailedWithPaymentCause() {

    // Arrange
    when(paymentGateway.charge(any()))
        .thenThrow(new PaymentException("Card declined"));

    // Act
    OrderFailedException ex =
        assertThrows(OrderFailedException.class,
            () -> orderService.placeOrder(request)
        );

    // Assert exception
    assertThat(ex.getMessage())
        .isEqualTo("Order failed due to payment error");

    // Assert CAUSE (chained exception)
    assertNotNull(ex.getCause());
    assertInstanceOf(PaymentException.class, ex.getCause());
    assertThat(ex.getCause().getMessage())
        .isEqualTo("Card declined");
}
```

---

### `assertThrows` with subclass exceptions

```java
// Exception hierarchy:
// Throwable
//   └── Exception
//         └── RuntimeException
//               └── IllegalArgumentException
//                     └── InvalidAgeException (custom)

@Test
void shouldUnderstandSubclassBehavior() {

    UserService service = new UserService();

    // assertThrows PASSES for exact type
    assertThrows(InvalidAgeException.class,
        () -> service.createUser("Alice", -1)
    ); // ✅ exact match

    // assertThrows ALSO PASSES for parent class!
    assertThrows(IllegalArgumentException.class,
        () -> service.createUser("Alice", -1)
    ); // ✅ passes — InvalidAgeException IS-A IllegalArgumentException

    assertThrows(RuntimeException.class,
        () -> service.createUser("Alice", -1)
    ); // ✅ passes — InvalidAgeException IS-A RuntimeException

    // Use assertThrowsExactly if subclass should NOT match:
    assertThrowsExactly(InvalidAgeException.class,
        () -> service.createUser("Alice", -1)
    ); // ✅ passes ONLY for exact InvalidAgeException
       // ❌ fails for IllegalArgumentException or RuntimeException
}
```

---

### `assertThrows` with multiple exception scenarios — `@ParameterizedTest`

```java
@ParameterizedTest
@MethodSource("invalidUserInputs")
void shouldThrowForAllInvalidInputs(String name, int age,
                                     String expectedMessage) {
    // Act
    IllegalArgumentException ex =
        assertThrows(IllegalArgumentException.class,
            () -> userService.createUser(name, age)
        );

    // Assert message
    assertThat(ex.getMessage()).isEqualTo(expectedMessage);
}

// Test data provider
static Stream<Arguments> invalidUserInputs() {
    return Stream.of(
        Arguments.of(null,    25,  "Name cannot be blank"),
        Arguments.of("",      25,  "Name cannot be blank"),
        Arguments.of("Alice", -1,  "Age cannot be negative"),
        Arguments.of("Alice", 200, "Age is unrealistic")
    );
}
// Runs 4 separate tests — one per invalid input ✅
```

---

### Common mistakes

#### Mistake 1 — Exception thrown OUTSIDE lambda

```java
// ❌ WRONG — exception thrown BEFORE lambda
User user = null;
assertThrows(NullPointerException.class,
    () -> user.getName()  // NullPointerException here — inside lambda ✅
);

// ❌ WRONG — exception thrown OUTSIDE lambda
String result = someMethodThatThrows(); // throws HERE — not caught!
assertThrows(RuntimeException.class,
    () -> doSomethingWith(result)       // never reached!
);
```

#### Mistake 2 — Not using the returned exception

```java
// ❌ WRONG — throwing away the returned exception
assertThrows(IllegalArgumentException.class,
    () -> service.createUser("Alice", -1)
);
// Can't verify message — exception is lost!

// ✅ CORRECT — capture and verify
IllegalArgumentException ex =
    assertThrows(IllegalArgumentException.class,
        () -> service.createUser("Alice", -1)
    );
assertThat(ex.getMessage()).isEqualTo("Age cannot be negative");
```

#### Mistake 3 — Too broad exception type

```java
// ❌ WRONG — too broad, catches everything
assertThrows(Exception.class,
    () -> service.createUser("Alice", -1)
);
// Passes even if NullPointerException thrown by a BUG!
// You wanted IllegalArgumentException but got NPE — bug hidden!

// ✅ CORRECT — specific exception type
assertThrows(IllegalArgumentException.class,
    () -> service.createUser("Alice", -1)
);
// Only passes for exactly this exception (or its subclasses)
```

---

### `assertThrows` vs AssertJ `assertThatThrownBy`

```java
// JUnit assertThrows — good
IllegalArgumentException ex =
    assertThrows(IllegalArgumentException.class,
        () -> service.createUser("Alice", -1)
    );
assertThat(ex.getMessage()).isEqualTo("Age cannot be negative");


// AssertJ assertThatThrownBy — more fluent, more readable
assertThatThrownBy(() -> service.createUser("Alice", -1))
    .isInstanceOf(IllegalArgumentException.class)
    .hasMessage("Age cannot be negative")
    .hasMessageContaining("negative")
    .hasMessageStartingWith("Age")
    .getCause() // if checking cause
        .isInstanceOf(SomeOtherException.class);

// Both are valid — AssertJ is more fluent for complex checks
// assertThrows is standard JUnit — no extra dependency needed
```

---

### Complete summary

```
assertThrows(ExceptionType.class, () -> { code })

PASSES when:
  ✅ Exact exception type thrown
  ✅ Subclass of exception type thrown

FAILS when:
  ❌ No exception thrown at all
  ❌ Different unrelated exception thrown

RETURNS:
  → The thrown exception object
  → Use it to verify message, cause, custom fields

COMPARE WITH:
  assertThrowsExactly → NO subclass allowed, exact type only
  assertDoesNotThrow  → opposite, verifies NO exception
  assertThatThrownBy  → AssertJ version, more fluent chaining
```

---

### Bottom line

> `assertThrows()` is the standard way to test that your code **throws the right exception** in the right situation
>
> Always **capture the returned exception** and verify its **message, cause, and custom fields** — not just the type
>
> Use **specific exception types** — never use `Exception.class` — it's too broad and hides real bugs
>
> One test per exception scenario — test null input, blank input, negative value separately — keeps tests **focused and easy to debug**













---

### 8. `assertThrowsExactly()` — checks EXACT exception type

```java
@Test
void shouldThrowExactExceptionType() {

    UserService service = new UserService();

    // assertThrows — passes for SUBCLASS too
    assertThrows(RuntimeException.class,
        () -> service.findById(-1L)
    ); // ✅ passes even if IllegalArgumentException is thrown
       // because IllegalArgumentException extends RuntimeException

    // assertThrowsExactly — MUST be EXACT type, no subclasses!
    assertThrowsExactly(IllegalArgumentException.class,
        () -> service.findById(-1L)
    ); // ✅ passes ONLY if EXACTLY IllegalArgumentException
       // ❌ fails if RuntimeException or any other subclass thrown
}
```

Key difference:

```
assertThrows(RuntimeException.class, ...)
    ✅ passes if: RuntimeException thrown
    ✅ passes if: IllegalArgumentException thrown (subclass!)
    ✅ passes if: NullPointerException thrown (subclass!)

assertThrowsExactly(RuntimeException.class, ...)
    ✅ passes if: EXACTLY RuntimeException thrown
    ❌ fails  if: IllegalArgumentException thrown (even though subclass)
    ❌ fails  if: NullPointerException thrown
```

---

### 9. `assertDoesNotThrow()` — checks NO exception thrown

```java
@Test
void shouldNotThrowForValidInput() {

    // Arrange
    UserService service = new UserService();

    // Assert — valid input should NOT throw any exception
    assertDoesNotThrow(
        () -> service.createUser("Alice", 25)
    ); // ✅ PASSES if no exception thrown
       // ❌ FAILS if ANY exception thrown

    // Failure message if exception thrown:
    // expected: no exception to be thrown
    // but was: IllegalArgumentException: ...
}
```

With return value:

```java
@Test
void shouldCreateUserWithoutException() {

    // assertDoesNotThrow can return the value!
    User user = assertDoesNotThrow(
        () -> userService.createUser("Alice", 25)
    );

    // Now assert on the returned user
    assertNotNull(user);
    assertThat(user.getName()).isEqualTo("Alice");
    // Clean — no try/catch needed! ✅
}
```

---

### 10. `assertTimeout()` — checks code finishes within time

```java
@Test
void shouldCompleteWithinTimeout() {

    // Assert — code must finish within 2 seconds
    assertTimeout(Duration.ofSeconds(2), () -> {

        // Your code here
        orderService.processOrder(orderId);

    }); // ✅ PASSES if finishes in < 2 seconds
        // ❌ FAILS if takes > 2 seconds

    // Key behavior:
    // Lets the code FINISH even if it exceeds timeout
    // THEN reports failure after completion
    // Does NOT interrupt execution!

    // Failure message:
    // execution exceeded timeout of 2000 ms by 500 ms
}
```

With return value:

```java
@Test
void shouldReturnResultWithinTimeout() {

    // assertTimeout returns the result!
    String result = assertTimeout(
        Duration.ofMillis(500),
        () -> paymentService.generateToken()
    );

    // Assert on returned value
    assertThat(result).isNotBlank();
    assertThat(result).hasSize(32);
}
```

---

### 11. `assertTimeoutPreemptively()` — STOPS code if timeout exceeded

```java
@Test
void shouldStopExecutionIfTooSlow() {

    // Assert — code STOPPED if exceeds 2 seconds
    assertTimeoutPreemptively(Duration.ofSeconds(2), () -> {

        orderService.processHeavyBatchJob();

    }); // ✅ PASSES if finishes in < 2 seconds
        // ❌ FAILS AND STOPS if takes > 2 seconds

    // Key difference from assertTimeout:
    // INTERRUPTS execution immediately when timeout exceeded!
    // Code does NOT continue running after timeout
}
```

Key difference:

```
assertTimeout()
    → code runs to completion THEN checks time
    → good when: cleanup/finally blocks must complete
    → code: [====runs====|timeout exceeded|====still runs====] then FAIL

assertTimeoutPreemptively()
    → code STOPS immediately when timeout hit
    → good when: preventing infinite loops or very long operations
    → code: [====runs====|timeout exceeded|STOPPED] FAIL immediately

⚠️ Warning: assertTimeoutPreemptively runs in separate thread
   → can cause issues with ThreadLocal (Spring's @Transactional!)
   → use assertTimeout for Spring integration tests
```

---

### 12. `assertAll()` — run ALL assertions, report ALL failures

```java
@Test
void shouldVerifyAllUserFieldsTogether() {

    // Arrange
    User user = userService.createUser("Alice", 25, "alice@gmail.com");

    // Assert — ALL assertions run even if first one fails!
    assertAll("Complete user validation",

        () -> assertNotNull(user,
            "User must not be null"),

        () -> assertThat(user.getName())
            .isEqualTo("Alice"),

        () -> assertThat(user.getAge())
            .isEqualTo(25),

        () -> assertThat(user.getEmail())
            .contains("@"),

        () -> assertThat(user.isActive())
            .isTrue(),

        () -> assertNotNull(user.getId(),
            "ID must be generated"),

        () -> assertNotNull(user.getCreatedAt(),
            "Timestamp must be set")
    );

    // WITHOUT assertAll:
    // If assertion 1 fails → assertions 2,3,4,5 NEVER run
    // You fix assertion 1, run again → assertion 2 fails
    // One failure visible at a time — SLOW debugging!

    // WITH assertAll:
    // ALL 7 assertions run regardless
    // ALL failures reported together:
    // Multiple failures (3):
    // 1) expected "Alice" but was "Bob"
    // 2) expected 25 but was 30
    // 3) expected true but was false
    // Fix ALL at once! ✅
}
```

Nested `assertAll` — grouped validations:

```java
@Test
void shouldVerifyOrderCompletely() {

    Order order = orderService.placeOrder(request);

    assertAll("Complete order validation",

        // Group 1: basic order fields
        () -> assertAll("Order basic fields",
            () -> assertNotNull(order.getId()),
            () -> assertNotNull(order.getCreatedAt()),
            () -> assertThat(order.getStatus()).isEqualTo("CONFIRMED")
        ),

        // Group 2: user details
        () -> assertAll("Order user details",
            () -> assertNotNull(order.getUserId()),
            () -> assertThat(order.getUserName()).isEqualTo("Alice")
        ),

        // Group 3: payment details
        () -> assertAll("Order payment details",
            () -> assertThat(order.getAmount()).isGreaterThan(0),
            () -> assertNotNull(order.getTransactionId()),
            () -> assertThat(order.getPaymentStatus()).isEqualTo("PAID")
        )
    );
}
```

---

### Summary table

| Assertion | Checks | Returns |
|---|---|---|
| `assertSame(a,b)` | `a == b` (same reference) | void |
| `assertNotSame(a,b)` | `a != b` (different reference) | void |
| `assertNull(a)` | `a == null` | void |
| `assertNotNull(a)` | `a != null` | void |
| `fail(msg)` | always fails | void |
| `assertInstanceOf(T, obj)` | `obj instanceof T` | typed T object |
| `assertThrows(T, code)` | T or subclass thrown | the exception |
| `assertThrowsExactly(T, code)` | EXACTLY T thrown | the exception |
| `assertDoesNotThrow(code)` | no exception thrown | return value |
| `assertTimeout(dur, code)` | finishes in time (lets finish) | return value |
| `assertTimeoutPreemptively(dur, code)` | finishes in time (stops it) | return value |
| `assertAll(executables...)` | all assertions run | void |

---

### Bottom line

> Each assertion has a specific purpose — use the right one for the right job
>
> `assertThrows` vs `assertThrowsExactly` — use Exactly when subclasses must NOT be accepted
>
> `assertTimeout` vs `assertTimeoutPreemptively` — use Preemptively to stop runaway code, but avoid it in Spring tests with `@Transactional`
>
> `assertAll` is your best friend for validating **multiple fields at once** — always shows ALL failures together, not just the first one







