### What is Unit Testing?

> It's a testing method where an **individual unit** (like a method or function) is tested **in isolation**.

```
Test:
When calculateMultiple(4, 2) is called, it should return 8.
```
We will be using Junit for unit testing and Mockito for mocking  and assert4j for asserts. Junit provide basic assertions but we need complex asserations and fluent api which is provided by assert4j.

![alt text](image.png)

- The goal is to test it in isolation — meaning we need to test the functionality of `calculateMultiply()` only, **not** any other method's functionality.
- In the example below, `calculateMultiply()` calls `NumberUtils` class's `multiply()` method, so we don't want to test `NumberUtils.multiply()`.
- So we **mock** this call `numUtil.multiply(a, b)`.

> **Mocking** means: instead of invoking the real method, we tell the test framework — "when this method is called, just return this predefined value without executing the actual logic."

```java
public class Calculator {
    private NumberUtils numUtil;

    public Calculator(NumberUtils numUtil) {
        this.numUtil = numUtil;
    }

    public int calculateMultiply(int a, int b) {
        // calling another class function
        return numUtil.multiply(a, b);
    }
}

public class NumberUtils {
    public int multiply(int a, int b) {
        return a * b;
    }
}
```

Mock here means when we call `numUtils.multiply(2,4)` return `8`.

---

### Advantages of Unit Testing

> It's a **"FIRST LINE OF DEFENCE"**

**Early Bug detection** — If a method returns the wrong output (e.g. `14` instead of `12`), you catch it while coding — not after release.

**Refactor Confidently** — When you change code, unit test cases warn you if something breaks — so you don't break features silently.

**Documentation** — Unit test cases show how a method behaves — no need to read all the code to understand what it does.

**Save Cost** — Fixing bugs during development is way cheaper than fixing them after the app goes live.

---

### Unit vs Integration vs Functional Testing

![alt text](image-1.png)

**Unit** — we've already seen above, it's testing of an individual unit (or method).

**Unit testing** is the most granular level — you test a single class or method in complete isolation, mocking all external dependencies. The goal is to verify that one unit of logic works correctly on its own.

---

### Integration Testing

> Testing of **multiple modules or components together**, to test if the system behaves properly when wired together.

**Integration testing** verifies that multiple components work correctly together. (Please read that line again: the goal is **not** to test end-to-end user behavior — the goal is just to check if multiple modules or the system, when wired together, achieve the intended behavior or not.)

**Use case 1: Wired multiple modules of 1 component**

```
Service -> Repository -> DB
```

- You invoke a method in the Service layer
- Verify that Spring Dependency Injection works correctly
- The flow should reach the database layer
- Your test can assert that data was persisted properly into the DB

Stub or mock the external service calls, as the goal is to test multiple modules of **1 component only**, not multiple microservices.

**Use case 2: Integration with multiple components** (not all the components present in the system)

```
Service A -> Service B
```
- Validate that Service A successfully calls Service B via REST (`RestTemplate`, Feign, etc.)
- We can test things like response mapping, HTTP error handling, etc.

```
Service A -> publish to Kafka
```
- Validate that a message is successfully published to the Kafka topic

```
Service A -> send mail
```
- Validate that the email payload is built and sent correctly

---

### Functional Testing

> **End to end testing** of a feature's business functionality, verifying it works as expected.

**Use case 1** — For 1 component, doing functional testing. End-to-end testing from our client's perspective:

```
API -> Controller -> Service (invokes Real external service call (REST call)) -> DB
```

**Use case 2** — End-to-end testing from the end user's perspective, i.e. how the end user interacts with the system (from the UI). May include frontend, backend, DB, Kafka, etc.

All components involved in a feature need to be deployed on a stage/QA environment, and then testing is done:

![alt text](image-2.png)

---

### Frameworks that Support Unit Testing

![alt text](image-3.png)

JUnit with Mockito is the most popular combination, and we'll use it in upcoming videos.
