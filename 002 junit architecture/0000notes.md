### JUnit4: Monolithic Architecture

This monolithic jar is responsible for **everything**:

- Test Discovery
- Test Execution
- Annotations
- Reporting
- etc.

![alt text](image.png)

> JUnit4 is currently in **"Maintenance Mode"**, and it's recommended to use JUnit5 for new development.

---

### JUnit5: Composed of Multiple Modules

![alt text](image-1.png)

Let's understand the complete architecture flow:

![alt text](image-2.png)

- **JUnit Platform** — the foundation layer. Exposes `junit-platform-engine`, `junit-platform-launcher`, and integration points (`junit-platform-sunfire-provider` for Maven, `junit-platform-gradle-plugin` for Gradle, `junit-platform-console` for CLI) so build tools, IDEs, and the command line can all talk to it.
- **JUnit Jupiter** — the Test Engine for JUnit5 test cases.
- **JUnit Vintage** — the Test Engine that runs old JUnit3/4 test cases on the new platform.
- **Other Test Engine** — any other framework (e.g. Cucumber) can plug in the same way.

---

### Step 1: Running a test case through IntelliJ

Say we run all the tests in a particular package, class, or a single test case:

![alt text](image-3.png)

---

### Step 2: IntelliJ directly talks to the Launcher, and initiates the `discover()` phase

Since IntelliJ talks directly with the Launcher, IntelliJ has the code to invoke the JUnit Launcher.

![alt text](image-4.png)

> The Launcher has 2 important methods:
> 1. `TestPlan discover(LauncherDiscoveryRequest discoveryRequest)`
> 2. `void execute(TestPlan testPlan, TestExecutionListener ...listeners)`

If we debug the call hierarchy, we can see the `JUnit5IdeaTestRunner` class — this is IntelliJ's framework code. It creates the `DiscoveryRequest` and invokes the JUnit Platform Launcher's `discover()` API:

![alt text](image-5.png)

So internally, IntelliJ's `JUnit5IdeaTestRunner` might be doing:

```java
// Specify what tests to run (packages, classes etc.)
LauncherDiscoveryRequest request = LauncherDiscoveryRequestBuilder.request()
        .selectors(selectPackage("com.conceptandcoding.learningspringboot"))
        .build();

// Creates an instance of Launcher
Launcher launcher = LauncherFactory.create();

// Invokes discover() method of JUnit Launcher
TestPlan testPlan = launcher.discover(request);
```

---

### Step 3: Launcher coordinates the `discover()` request across all registered Test Engines

The Launcher acts as a **coordinator** and passes the `discover()` request to all registered Test Engines, so each can discover and filter out their own test cases:

![alt text](image-6.png)

**`junit-platform-engine`**

> Provides an **SPI (Service Provider Interface)**. Exposes an interface which different test engines (Vintage, Jupiter, etc.) implement to integrate with the JUnit Platform:

```java
interface TestEngine {

    TestDescriptor discover(EngineDiscoveryRequest request, UniqueId id);

    void execute(ExecutionRequest request);
}
```

**`junit-platform-launcher`**

Launcher methods:
1. `TestPlan discover(LauncherDiscoveryRequest discoveryRequest)`
2. `void execute(TestPlan testPlan, TestExecutionListener ...listeners)`

**In the `discover()` phase:**

- The Launcher receives a `LauncherDiscoveryRequest` with a selector (package name, class name, etc.)
- The Launcher delegates this request to all the Test Engines (Jupiter, Vintage, other frameworks, etc.)

```java
// Launcher iterates and calls the discover method for each Test Engine.
for (TestEngine engine : testEngines) {
    TestDescriptor descriptor = engine.discover(...);
    root.addChild(descriptor);
}
```

---

### Each Test Engine discovers and filters its own framework's test cases

Say in a package we have multiple framework test cases — JUnit4, JUnit5, Cucumber, etc.

The JUnit5 (Jupiter) Test Engine takes out its own framework test cases:

```
Class A:
  - Method1
  - Method2

Class B:
  - Method1
  - Method2
  - Method3
```

The JUnit4 (Vintage) Test Engine similarly takes out its own framework test cases:

```
Class C:
  - Method1
  - Method2

Class D:
  - Method1
```

Once the Launcher gets the response from all the Test Engines, it creates a single `TestPlan` out of it:

```
TestPlan =
  Jupiter Framework test cases:
    Class A: Method1, Method2
    Class B: Method1, Method2, Method3

  Vintage Framework test cases:
    Class C: Method1, Method2
    Class D: Method1
```

...and returns it back to the caller (IntelliJ).

---

### Step 4: IntelliJ calls `execute()` on the Launcher

![alt text](image-7.png)

**During execution**, `launcher.execute(TestPlan, ...)` is called:

```java
LauncherDiscoveryRequest request = LauncherDiscoveryRequestBuilder.request()
        .selectors(selectPackage("com.conceptandcoding.learningspringboot"))
        .build();

Launcher launcher = LauncherFactory.create();
TestPlan testPlan = launcher.discover(request);

launcher.execute(request);
```

---

### Step 5: Launcher coordinates execution across each Test Engine

Again, the Launcher acts as a coordinator and tells each Test Engine to run **only its own tests**.

The `TestPlan` contains a **tree of `TestDescriptor`s**, each tied to its corresponding `TestEngine`:

```
TestPlan =
  Jupiter Framework test cases (passed to Jupiter Test Engine):
    Class A: Method1, Method2
    Class B: Method1, Method2, Method3

  Vintage Framework test cases (passed to Vintage Test Engine):
    Class C: Method1, Method2
    Class D: Method1
```

- Invokes `TestEngine.execute(ExecutionRequest)` for each engine, with only its relevant subtree.
- Each Test Engine returns a report of pass/fail results; the Launcher combines them and returns the final report back to IntelliJ.

![alt text](image-8.png)

![alt text](image-9.png)

---

### Key Advantages of this Architecture

- **Multiple test frameworks can be supported** — Jupiter, Vintage, or any custom `TestEngine`, all plugged in through the same Launcher/Platform.
- **Parallel migration** — new test cases can be written in JUnit5 while old JUnit4 test cases keep running as-is, since the Vintage Test Engine runs JUnit4 (legacy) test cases on the latest framework too.
- **Parallel test execution** is supported (at the Test Engine level, or within an engine).
- **Any new `TestEngine`** is automatically supported across Maven, Gradle, IntelliJ, Eclipse, etc. — thanks to the Launcher acting as a mediator.
