# Notes
>Note: Code in Spring Udemy Repository

We will test add method for now!!

We need to add maven depency Junit !!

Development Process Step-By-Step

1. Add Maven dependencies for JUnit

2. Create test package

3. Create unit test

4. Run unit test

![alt text](image.png)

>Note:Scope=test here means this dependency is only needed while testing not for normal execution flow!!

We have code to add in DemoUtils

![alt text](image-1.png)

![alt text](image-2.png)

![alt text](image-3.png)

TestMethod can have any name just annotate with `@Test`!!

![alt text](image-4.png)

## Junit assertions

![alt text](image-5.png)


### ✅ Common JUnit Assertions (JUnit 5)
|Method	|Description|
|-------|------------|
|assertEquals(expected, actual)	|Passes if expected == actual
|assertNotEquals(expected, actual)	|Passes if expected != actual
|assertTrue(condition)	|Passes if condition is true
|assertFalse(condition)	|Passes if condition is false
|assertNull(object)	|Passes if object is null
|assertNotNull(object)	|Passes if object is not null
|assertThrows()	|Passes if the specified exception is thrown
|assertAll()	|Groups multiple assertions and evaluates all of them
|assertSame(expected, actual)	|Passes if both refer to the same object (reference equality)
|assertNotSame(expected, actual)	|Passes if both refer to different objects

![alt text](image-6.png)

![alt text](image-7.png)

![alt text](image-8.png)

### Static import 

![alt text](image-9.png)

So for static method instead of calling by className just use static imports!!

![alt text](image-10.png)

can use even wildcard `*` to get all methods imported!!

### Restructured code

![alt text](image-11.png)

Now we want to test 

![alt text](image-12.png)

can use aseertNull 

![alt text](image-13.png)

## Multiple tests in one file 

![alt text](image-14.png)

## Code 

TEstClass Need not to be public

DemoUtilsTests.java

```java


import org.junit.jupiter.api.*;

import static org.junit.jupiter.api.Assertions.*;

class DemoUtilsTest {

    @Test
    void testEqualsAndNotEquals() {

        DemoUtils demoUtils = new DemoUtils();

        assertEquals(6, demoUtils.add(2, 4), "2+4 must be 6");
        assertNotEquals(6, demoUtils.add(1, 9), "1+9 must not be 6");
    }

    @Test
    void testNullAndNotNull() {

        DemoUtils demoUtils = new DemoUtils();

        String str1 = null;
        String str2 = "luv2code";

        assertNull(demoUtils.checkNull(str1), "Object should be null");
        assertNotNull(demoUtils.checkNull(str2), "Object should not be null");

    }

}
```
DemoUtils.java

```java

package com.luv2code.junitdemo;

import java.util.List;

public class DemoUtils {

    private String academy = "Luv2Code Academy";
    private String academyDuplicate = academy;
    private String[] firstThreeLettersOfAlphabet = {"A", "B", "C"};
    private List<String> academyInList = List.of("luv", "2", "code");

    public List<String> getAcademyInList() {
        return academyInList;
    }

    public String getAcademy() {
        return academy;
    }

    public String getAcademyDuplicate() {
        return academyDuplicate;
    }

    public String[] getFirstThreeLettersOfAlphabet() {
        return firstThreeLettersOfAlphabet;
    }

    public int add(int a, int b) {
        return a + b;
    }

    public int multiply(int a, int b) {
        return a * b;
    }

    public Object checkNull(Object obj) {

        if (obj != null) {
            return obj;
        }
        return null;

    }

    public Boolean isGreater(int n1, int n2) {
        if (n1 > n2) {
            return true;
        }
        return false;
    }

    public String throwException(int a) throws Exception {
        if (a < 0) {
            throw new Exception("Value should be greater than or equal to 0");
        }
        return "Value is greater than or equal to 0";
    }

    public void checkTimeout() throws InterruptedException {
        System.out.println("I am going to sleep");
        Thread.sleep(2000);
        System.out.println("Sleeping over");
    }

}

```

## Lifecycle methods

### BeforeEach

Whatever you want to do beforeEach TestCase

![alt text](image-15.png)

Similaryly we have After Each

![alt text](image-16.png)

Lifecycle Methods

• When developing tests, we may need to perform one-time operations

-  One-time set up before all tests

    - Get database connections, connect to remote servers ...

- One-time clean up after all tests

    - Release database connections, disconnect from remote servers ...

Then we have `BeforeAll()` and `AfterAll()`

![alt text](image-17.png)

## Code

testClass

```java

package com.luv2code.junitdemo;

import org.junit.jupiter.api.*;

import static org.junit.jupiter.api.Assertions.*;

class DemoUtilsTest {

    DemoUtils demoUtils;

    @BeforeEach
    void setupBeforeEach() {
        demoUtils = new DemoUtils();
        System.out.println("@BeforeEach executes before the execution of each test method");
    }

    @AfterEach
    void tearDownAfterEach() {
        System.out.println("Running @AfterEach");
        System.out.println();
    }

    @BeforeAll
    static void setupBeforeEachClass() {
        System.out.println("@BeforeAll executes only once before all test methods execution in the class");
    }

    @AfterAll
    static void tearDownAfterAll() {
        System.out.println("@AfterAll executes only once after all test methods execution in the class");
    }


    @Test
    void testEqualsAndNotEquals() {

        System.out.println("Running test: testEqualsAndNotEquals");

        assertEquals(6, demoUtils.add(2, 4), "2+4 must be 6");
        assertNotEquals(6, demoUtils.add(1, 9), "1+9 must not be 6");
    }

    @Test
    void testNullAndNotNull() {

        System.out.println("Running test: testNullAndNotNull");

        String str1 = null;
        String str2 = "luv2code";

        assertNull(demoUtils.checkNull(str1), "Object should be null");
        assertNotNull(demoUtils.checkNull(str2), "Object should not be null");

    }

}

```

Output:

![alt text](image-18.png)

## Custom Display Names


• Currently the method names are listed in test results

• We'd like to give custom display names

• Provide a more descriptive name for the test

• Include spaces, special characters: Test for Equality to support JIRA #123

• Useful for sharing test reports with project management and non-techies

can be given by annotation `@DisplayName`

Custom display name with spaces, special characters and emojis.

Useful for test reports in IDE or external test runner

![alt text](image-19.png)

![alt text](image-20.png)

![alt text](image-21.png)

![alt text](image-22.png)

![alt text](image-23.png)

## Assertions for same 

![alt text](image-24.png)

Code to test

```java
package com.luv2code.junitdemo;

public class DemoUtils {

    private String academy = "Luv2Code Academy";
    private String academyDuplicate = academy;

    public String getAcademy() {
        return academy;
    }

    public String getAcademyDuplicate() {
        return academyDuplicate;
    }
}
```
![alt text](image-25.png)

```java

    @DisplayName("Same and Not Same")
    @Test
    void testSameAndNotSame() {

        String str = "luv2code";

        assertSame(demoUtils.getAcademy(), demoUtils.getAcademyDuplicate(), "Objects should refer to same object");
        assertSame(str, demoUtils.getAcademy(), "Objects should not refer to same object");
    }

 ```

 If a test-case is not passed then only Optional message is dipalyed!!

 ![alt text](image-26.png)

 ## Assertions for true 

 ![alt text](image-27.png)

 ## Assert for array 

 assertArrayEquals--> Assert that both object arrays are deeply equal

 **assertArrayEquals()** is used to check whether two arrays are equal in terms of both content and order.

 ![alt text](image-28.png)

 ```java

 String[] expected = {"java", "python"};
String[] actual = {"python", "java"};

assertArrayEquals(expected, actual); // fails due to different order
```

assertIterableEquals-->Assert that both object iterables are deeply equal

assertIterableEquals is used to compare two Iterable objects (like List, Set, etc.) to check if they contain equal elements in the same order.

![alt text](image-29.png)

assertLinesMatch-->Assert that both lists of strings match

![alt text](image-30.png)

assertLinesMatch() is a JUnit 5 assertion used to compare two lists of strings line by line, with support for regex patterns in the expected list. It's especially useful when:

- You want to check output logs or multiline strings.

- You need to allow some flexibility in matching (e.g., variable parts like timestamps).

## Assertions: Throws and Timeout

assertThrows-->Assert that an executable throws an exception of expected type

![alt text](image-31.png)

![alt text](image-32.png)



















