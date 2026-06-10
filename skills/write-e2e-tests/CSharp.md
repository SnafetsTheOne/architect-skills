# E2E in C\#

## Stack

Follow what the repo already uses; these are the defaults when there is no signal.

| Concern         | Default                                                          |
| --------------- | ---------------------------------------------------------------- |
| Test framework  | xUnit                                                            |
| Browser         | `Microsoft.Playwright`                                           |
| Mocked services | `WireMock.Net`                                                   |
| Generators      | hand-rolled static `Generate()` factories (Bogus if data is rich) |
| DB access       | the app's own EF Core model if referencable; otherwise Dapper    |

After the first build, install browsers: `pwsh bin/Debug/net8.0/playwright.ps1 install chromium`.

## Layout

Folder names follow the layer roles — rename to fit the project.

```text
e2e/
├── CLAUDE.md
├── E2E.csproj
├── Plumbing/
│   ├── Scenario.cs      Given/When/Then builder (below)
│   ├── E2EFixture.cs    starts Playwright + WireMock, resolves BaseUrl + db
│   └── E2ETest.cs       base class exposing Given(...)
├── Primitives/
│   ├── Database.cs      connection / DbContext factory
│   ├── MockServer.cs    WireMock stubbing helpers
│   └── Generators/      Customer.cs, Order.cs, …
├── Abstractions/
│   ├── Db.cs            db.Customers, db.Orders, …
│   ├── Ui.cs            ui.Login, ui.PlaceOrder, ui.Cart, …
│   └── Mocks.cs         mocks.PaymentProvider, …
└── Tests/
    └── OrderTests.cs
```

## Plumbing — `Scenario.cs`

This exact overload shape is verified to compile and resolve; keep it.

```csharp
using System.Runtime.CompilerServices;

public sealed class Scenario(E2EFixture fx)
{
    private readonly List<(string Keyword, string Text, Func<Task> Run)> _steps = [];
    private readonly Db _db = new(fx);
    private readonly Mocks _mocks = new(fx);
    private Ui? _ui;
    private static readonly TimeSpan AssertTimeout = TimeSpan.FromSeconds(10);

    public Scenario Given(Func<Db, Task> step, [CallerArgumentExpression(nameof(step))] string text = "")
        => Add("Given", text, () => step(_db));
    public Scenario Given(Func<Mocks, Task> step, [CallerArgumentExpression(nameof(step))] string text = "")
        => Add("Given", text, () => step(_mocks));
    public Scenario When(Func<Ui, Task> step, [CallerArgumentExpression(nameof(step))] string text = "")
        => Add("When", text, () => step(_ui!));
    public Scenario When(Func<Db, Task> step, [CallerArgumentExpression(nameof(step))] string text = "")
        => Add("When", text, () => step(_db));
    // predicates — poll while false
    public Scenario Then(Func<Db, bool> check, [CallerArgumentExpression(nameof(check))] string text = "")
        => Add("Then", text, () => Eventually(() => Task.FromResult(check(_db)), text));
    public Scenario Then(Func<Db, Task<bool>> check, [CallerArgumentExpression(nameof(check))] string text = "")
        => Add("Then", text, () => Eventually(() => check(_db), text));
    public Scenario Then(Func<Ui, Task<bool>> check, [CallerArgumentExpression(nameof(check))] string text = "")
        => Add("Then", text, () => Eventually(() => check(_ui!), text));

    // assertions (any library) — poll while throwing
    public Scenario Then(Action<Db> assertion, [CallerArgumentExpression(nameof(assertion))] string text = "")
        => Add("Then", text, () => EventuallyAssert(() => { assertion(_db); return Task.CompletedTask; }, text));
    public Scenario Then(Func<Db, Task> assertion, [CallerArgumentExpression(nameof(assertion))] string text = "")
        => Add("Then", text, () => EventuallyAssert(() => assertion(_db), text));
    public Scenario Then(Func<Ui, Task> assertion, [CallerArgumentExpression(nameof(assertion))] string text = "")
        => Add("Then", text, () => EventuallyAssert(() => assertion(_ui!), text));

    public TaskAwaiter GetAwaiter() => RunAsync().GetAwaiter();

    private Scenario Add(string keyword, string text, Func<Task> run)
    {
        _steps.Add((keyword, text, run));
        return this;
    }

    private static async Task Eventually(Func<Task<bool>> check, string text)
    {
        var deadline = DateTime.UtcNow + AssertTimeout;
        while (!await check())
        {
            if (DateTime.UtcNow > deadline)
                throw new ScenarioFailedException($"Then {text} — still false after {AssertTimeout.TotalSeconds}s");
            await Task.Delay(250);
        }
    }

    private static async Task EventuallyAssert(Func<Task> assertion, string text)
    {
        var deadline = DateTime.UtcNow + AssertTimeout;
        while (true)
        {
            try { await assertion(); return; }
            catch (Exception e)
            {
                if (DateTime.UtcNow > deadline)
                    throw new ScenarioFailedException($"Then {text} — still failing after {AssertTimeout.TotalSeconds}s: {e.Message}", e);
                await Task.Delay(250);
            }
        }
    }

    private async Task RunAsync()
    {
        await using var context = await fx.Browser.NewContextAsync(new() { BaseURL = fx.BaseUrl });
        _ui = new Ui(await context.NewPageAsync());
        foreach (var (keyword, text, run) in _steps)
        {
            try { await run(); }
            catch (Exception e)
            {
                var shot = await TryScreenshot();
                var message = e is ScenarioFailedException ? e.Message : $"{keyword} {text} — threw {e.GetType().Name}";
                throw new ScenarioFailedException(shot is null ? message : $"{message}\nscreenshot: {shot}", e);
            }
        }
    }

    private async Task<string?> TryScreenshot()
    {
        try
        {
            var path = Path.Combine("artifacts", $"{Guid.NewGuid():N}.png");
            return await _ui!.Page.ScreenshotAsync(new() { Path = path, FullPage = true });
        }
        catch { return null; }
    }
}

public sealed class ScenarioFailedException(string message, Exception? inner = null)
    : Exception(message, inner);
```

Why this shape — keep these properties when extending it:

- **Untyped lambdas resolve** (`db => …`, `ui => …`) because the body only
  binds against one facade. An ambiguity needs the *whole body* to bind on
  both — a `ui.Orders` page next to a `db.Orders` table facade only clashes
  when the chained members also match. Keeping facade roots thin (one property
  per domain area, actions on the area objects) makes that rare; when it does
  happen, type the lambda explicitly: `(Db db) => …`.
- **All facade actions return `Task`** (wrap sync work in `Task.CompletedTask`)
  so the overload set stays small and unambiguous; checks return `bool` or
  `Task<bool>`.
- **`Then` takes predicates or assertions, and both poll** — When steps trigger
  async backend work, so asserting on the first read races it. Predicates poll
  while false; assertion steps poll while throwing, so any assertion library
  works (xUnit `Assert`, FluentAssertions, Shouldly — the harness only sees
  throw vs return) and its expected-vs-actual message survives into the timeout
  failure. Keep the `Task<bool>` predicate overloads even if unused: they catch
  `db => db.Orders.Contains(…)`, which would otherwise bind to an assertion
  overload, discard the boolean, and silently pass.
- **`[CallerArgumentExpression]`** puts the lambda's source text in the failure
  message: `Then db.Orders.Contains(order, customer) — still false after 10s`.
- **`GetAwaiter()` runs the recorded steps**, so a test is one awaited chain —
  no terminal `.Run()` to forget.

## Plumbing — fixture and test base

```csharp
[CollectionDefinition("e2e")]
public sealed class E2ECollection : ICollectionFixture<E2EFixture>;

public sealed class E2EFixture : IAsyncLifetime
{
    public IBrowser Browser { get; private set; } = null!;
    public WireMockServer MockServer { get; private set; } = null!;
    public string BaseUrl { get; } = Environment.GetEnvironmentVariable("E2E_BASE_URL") ?? "http://localhost:5000";
    public string ConnectionString { get; } = Environment.GetEnvironmentVariable("E2E_DB")
        ?? "Host=localhost;Database=app;Username=e2e;Password=e2e";
    private IPlaywright _playwright = null!;

    public async Task InitializeAsync()
    {
        MockServer = WireMockServer.Start(9090); // app must point its outbound clients here
        _playwright = await Playwright.CreateAsync();
        Browser = await _playwright.Chromium.LaunchAsync(new() { Headless = true });
    }

    public async Task DisposeAsync()
    {
        await Browser.DisposeAsync();
        _playwright.Dispose();
        MockServer.Stop();
    }
}

[Collection("e2e")]
public abstract class E2ETest(E2EFixture fx)
{
    protected Scenario Given(Func<Db, Task> step, [CallerArgumentExpression(nameof(step))] string text = "")
        => new Scenario(fx).Given(step, text);
    protected Scenario Given(Func<Mocks, Task> step, [CallerArgumentExpression(nameof(step))] string text = "")
        => new Scenario(fx).Given(step, text);
}
```

How the app under test starts (docker compose, `dotnet run`, deployed URL) is
project-specific — ask the user, wire it into the fixture or compose file, and
record it under **Running** in the e2e `CLAUDE.md`.

## Facades, primitives, generators (skeleton)

Facade roots stay thin: one property per domain area (screens/flows under
`Ui`, entities under `Db`), actions on the area objects — grow by adding
areas, not root methods.

```csharp
// Abstractions/Ui.cs — clusters Playwright primitives into business actions
public sealed class Ui(IPage page)
{
    public IPage Page { get; } = page;
    public Cart Cart { get; } = new(page);

    public async Task Login(Customer customer)
    {
        await Page.GotoAsync("/login");
        await Page.GetByLabel("Email").FillAsync(customer.Email);
        await Page.GetByLabel("Password").FillAsync(customer.Password);
        await Page.GetByRole(AriaRole.Button, new() { Name = "Log in" }).ClickAsync();
    }
}

// Abstractions/Db.cs — inserts whole entity graphs, queries for assertions
public sealed class Db(E2EFixture fx)
{
    public CustomersFacade Customers { get; } = new(fx);
    public OrdersFacade Orders { get; } = new(fx);
}

public sealed class OrdersFacade(E2EFixture fx)
{
    public async Task<bool> Contains(Order order, Customer customer)
    {
        // fresh context per call — a cached one returns stale entities while Then polls
        await using var db = Database.NewContext(fx.ConnectionString);
        return await db.Orders.AnyAsync(o => o.Customer.Email == customer.Email
                                          && o.Items.Any(i => i.Sku == order.Sku));
    }
}

// Primitives/Generators/Customer.cs — e2e-side test data, not the app's entity
public sealed record Customer(string Email, string Password, string Name)
{
    public static Customer Generate()
    {
        var id = Guid.NewGuid().ToString("N")[..8];
        return new($"e2e-{id}@example.test", Guid.NewGuid().ToString("N"), $"E2E Customer {id}");
    }
}
```

## Example test

```csharp
public sealed class OrderTests(E2EFixture fx) : E2ETest(fx)
{
    [Fact]
    public async Task Customer_can_order_and_pay()
    {
        var customer = Customer.Generate();
        var order = Order.Generate();

        await Given(mocks => mocks.PaymentProvider.AcceptsAll())
            .Given(db => db.Customers.Add(customer))
            .When(ui => ui.Login(customer))
            .When(ui => ui.PlaceOrder(order))
            .Then(db => db.Orders.Contains(order, customer))
            .When(ui => ui.CheckoutAndPay(customer))
            .Then(ui => ui.Cart.IsEmpty())
            .Then(db => Assert.Equal("Ordered", db.Orders.StatusOf(order, customer)));
    }
}
```

The first two `Then`s are predicates; the last is an assertion — use whichever
reads better, preferring the assertion form when expected-vs-actual matters in
the failure message.

xUnit runs test classes in parallel by default — generated data is what keeps
them from colliding in the shared database.
