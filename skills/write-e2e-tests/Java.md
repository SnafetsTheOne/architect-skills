# E2E in Java

## Stack

Follow what the repo already uses; these are the defaults when there is no signal.

| Concern         | Default                                                              |
| --------------- | -------------------------------------------------------------------- |
| Test framework  | JUnit 5, Java 17+                                                    |
| Browser         | `com.microsoft.playwright:playwright` (sync API)                     |
| Mocked services | `org.wiremock:wiremock`                                              |
| Generators      | hand-rolled static `generate()` factories (Datafaker if data is rich) |
| DB access       | JDBC `DataSource` (or the app's JPA entities if referencable)        |

Build as a separate module (`e2e/` with its own `pom.xml` / Gradle project) so
app builds don't pay for it. Install browsers once:
`mvn exec:java -Dexec.mainClass=com.microsoft.playwright.CLI -Dexec.args="install chromium"`.

## Layout

Folder names follow the layer roles — rename to fit the project.

```text
e2e/
├── CLAUDE.md
├── pom.xml
└── src/test/java/e2e/
    ├── plumbing/        Scenario.java, Steps.java, E2ETest.java
    ├── primitives/      Database.java, MockServer.java, generators/
    ├── abstractions/    Db.java, Ui.java, Mocks.java
    └── tests/           OrderE2ETest.java
```

## Plumbing — `Steps.java` and `Scenario.java`

Two Java constraints shape this (the C# version needs neither — don't copy
its shape blindly in either direction):

- **Per-facade functional interfaces, not `Consumer<Db>`/`Consumer<Ui>`** —
  generic overloads erase to the same signature and won't compile.
- **Lambda parameters are explicitly typed** (`(Db db) -> …`) — Java picks
  overloads by parameter type only, never by whether the body would bind, so
  implicit lambdas are ambiguous.

```java
// Steps.java
@FunctionalInterface public interface DbStep   { void run(Db db); }
@FunctionalInterface public interface UiStep   { void run(Ui ui); }
@FunctionalInterface public interface MockStep { void run(Mocks mocks); }
@FunctionalInterface public interface DbCheck  { boolean test(Db db); }
@FunctionalInterface public interface UiCheck  { boolean test(Ui ui); }
```

This exact overload shape is verified to compile and resolve; keep it. Steps
execute eagerly — each call runs immediately and returns the scenario, so
there is no terminal `.run()` to forget.

```java
// Scenario.java
public final class Scenario {
    private static final Duration ASSERT_TIMEOUT = Duration.ofSeconds(10);
    private final Db db;
    private final Ui ui;
    private final Mocks mocks;
    private int stepNo;

    public Scenario(Db db, Ui ui, Mocks mocks) {
        this.db = db; this.ui = ui; this.mocks = mocks;
    }

    public Scenario given(DbStep step)   { return exec("Given", () -> step.run(db)); }
    public Scenario given(MockStep step) { return exec("Given", () -> step.run(mocks)); }
    public Scenario when(UiStep step)    { return exec("When",  () -> step.run(ui)); }
    public Scenario when(DbStep step)    { return exec("When",  () -> step.run(db)); }
    // predicates — poll while false
    public Scenario then(DbCheck check)  { return exec("Then",  () -> eventually(() -> check.test(db))); }
    public Scenario then(UiCheck check)  { return exec("Then",  () -> eventually(() -> check.test(ui))); }

    // assertions (any library) — poll while throwing
    public Scenario thenAssert(DbStep assertion) { return exec("Then", () -> eventuallyAssert(() -> assertion.run(db))); }
    public Scenario thenAssert(UiStep assertion) { return exec("Then", () -> eventuallyAssert(() -> assertion.run(ui))); }

    private Scenario exec(String keyword, Runnable body) {
        stepNo++;
        try { body.run(); }
        catch (AssertionError | RuntimeException e) {
            String shot = ui.tryScreenshot(); // null if the page is gone
            throw new AssertionError(keyword + " step #" + stepNo + " failed: " + e.getMessage()
                + (shot == null ? "" : "\nscreenshot: " + shot), e);
        }
        return this;
    }

    // When steps trigger async backend work; asserting on the first read races it.
    private static void eventually(BooleanSupplier check) {
        long deadline = System.currentTimeMillis() + ASSERT_TIMEOUT.toMillis();
        while (!check.getAsBoolean()) {
            if (System.currentTimeMillis() > deadline)
                throw new IllegalStateException("still false after " + ASSERT_TIMEOUT.toSeconds() + "s");
            sleep();
        }
    }

    private static void eventuallyAssert(Runnable assertion) {
        long deadline = System.currentTimeMillis() + ASSERT_TIMEOUT.toMillis();
        while (true) {
            try { assertion.run(); return; }
            catch (AssertionError | RuntimeException e) {
                if (System.currentTimeMillis() > deadline) throw e;
                sleep();
            }
        }
    }

    private static void sleep() {
        try { Thread.sleep(250); } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new IllegalStateException(e);
        }
    }
}
```

Java cannot capture the lambda's source text, so predicate failures carry only
the step number; when expected-vs-actual detail matters, use `thenAssert` with
the project's assertion library (AssertJ, JUnit assertions — the harness only
sees throw vs return, and the message survives into the timeout failure).

`thenAssert` is a separate method, not a `then` overload, on purpose: a
boolean-returning call like `db.orders().contains(…)` is also a valid void
statement, so `then(DbStep)` next to `then(DbCheck)` would be ambiguous — and
routing a predicate into a void step would discard the boolean and pass
silently.

## Plumbing — `E2ETest.java` base class

```java
public abstract class E2ETest {
    protected static Playwright playwright;
    protected static Browser browser;
    protected static WireMockServer mockServer;
    private final List<BrowserContext> contexts = new ArrayList<>();

    static String baseUrl() { return System.getenv().getOrDefault("E2E_BASE_URL", "http://localhost:8080"); }

    @BeforeAll
    static void startInfra() {
        mockServer = new WireMockServer(9090); // app must point its outbound clients here
        mockServer.start();
        playwright = Playwright.create();
        browser = playwright.chromium().launch(new BrowserType.LaunchOptions().setHeadless(true));
    }

    @AfterAll
    static void stopInfra() {
        browser.close();
        playwright.close();
        mockServer.stop();
    }

    @AfterEach
    void closeContexts() { contexts.forEach(BrowserContext::close); contexts.clear(); }

    protected Scenario given(DbStep step)   { return scenario().given(step); }
    protected Scenario given(MockStep step) { return scenario().given(step); }

    private Scenario scenario() {
        BrowserContext context = browser.newContext(new Browser.NewContextOptions().setBaseURL(baseUrl()));
        contexts.add(context);
        return new Scenario(new Db(Database.dataSource()),
                            new Ui(context.newPage()),
                            new Mocks(mockServer));
    }
}
```

How the app under test starts (docker compose, `mvn spring-boot:run`, deployed
URL) is project-specific — ask the user, wire it in, and record it under
**Running** in the e2e `CLAUDE.md`.

## Facades, primitives, generators (skeleton)

Facade roots stay thin: one method per domain area (screens/flows under `Ui`,
entities under `Db`), actions on the area objects — grow by adding areas, not
root methods.

```java
// abstractions/Ui.java — clusters Playwright primitives into business actions
public final class Ui {
    private final Page page;
    public Ui(Page page) { this.page = page; }
    public Cart cart() { return new Cart(page); }

    public void login(Customer customer) {
        page.navigate("/login");
        page.getByLabel("Email").fill(customer.email());
        page.getByLabel("Password").fill(customer.password());
        page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Log in")).click();
    }

    String tryScreenshot() {
        try {
            Path path = Path.of("artifacts", UUID.randomUUID() + ".png");
            page.screenshot(new Page.ScreenshotOptions().setPath(path).setFullPage(true));
            return path.toString();
        } catch (RuntimeException e) { return null; }
    }
}

// abstractions/Db.java — inserts whole entity graphs, queries for assertions
public final class Db {
    private final DataSource ds;
    public Db(DataSource ds) { this.ds = ds; }
    public Customers customers() { return new Customers(ds); }
    public Orders orders() { return new Orders(ds); }
}

public final class Orders {
    private final DataSource ds;
    public Orders(DataSource ds) { this.ds = ds; }

    public boolean contains(Order order, Customer customer) {
        // fresh query per call — then() polls this
        try (var con = ds.getConnection();
             var st = con.prepareStatement(
                 "select count(*) from orders o join customers c on c.id = o.customer_id "
                 + "where c.email = ? and o.sku = ?")) {
            st.setString(1, customer.email());
            st.setString(2, order.sku());
            try (var rs = st.executeQuery()) { return rs.next() && rs.getInt(1) > 0; }
        } catch (SQLException e) { throw new IllegalStateException(e); }
    }
}

// primitives/generators/Customer.java — e2e-side test data, not the app's entity
public record Customer(String email, String password, String name) {
    public static Customer generate() {
        String id = UUID.randomUUID().toString().substring(0, 8);
        return new Customer("e2e-" + id + "@example.test", UUID.randomUUID().toString(), "E2E Customer " + id);
    }
}
```

## Example test

```java
class OrderE2ETest extends E2ETest {
    @Test
    void customerCanOrderAndPay() {
        var customer = Customer.generate();
        var order = Order.generate();

        given((Mocks mocks) -> mocks.paymentProvider().acceptsAll())
            .given((Db db) -> db.customers().add(customer))
            .when((Ui ui) -> ui.login(customer))
            .when((Ui ui) -> ui.placeOrder(order))
            .then((Db db) -> db.orders().contains(order, customer))
            .when((Ui ui) -> ui.checkoutAndPay(customer))
            .then((Ui ui) -> ui.cart().isEmpty())
            .thenAssert((Db db) -> assertThat(db.orders().statusOf(order, customer)).isEqualTo("Ordered"));
    }
}
```

The `then`s are polled predicates; `thenAssert` is a polled assertion — prefer
it when expected-vs-actual matters in the failure message.

Generated data is what lets tests run in parallel against the shared database
— never hardcode emails, SKUs, or names.
