---
name: write-e2e-tests
description: >-
  Use when the user wants to write, add, or restructure end-to-end (E2E) tests,
  or set up an E2E test suite — BDD-style Given/When/Then scenario tests,
  browser/UI automation (Playwright), layered test infrastructure, test-data
  generators, or mocked external services. Also use when adding tests to an
  e2e folder that has its own CLAUDE.md.
---

# Layered E2E Tests

Behavior-driven testing (BDD) with the scenario written directly in code —
Given/When/Then without the Gherkin feature-file indirection; the test itself
is the readable artifact. A test reads like the business scenario it verifies:
no selectors, SQL, or HTTP in sight; everything technical lives in a lower
layer, written once.

```csharp
var customer = Customer.Generate();
var order = Order.Generate();

await Given(db => db.Customers.Add(customer))
    .When(ui => ui.Login(customer))
    .When(ui => ui.PlaceOrder(order))
    .Then(db => db.Orders.Contains(order, customer))
    .When(ui => ui.CheckoutAndPay(customer))
    .Then(ui => ui.Cart.IsEmpty())
    .Then(db => db.Orders.StatusOf(order, customer) == "Ordered");
```

This example carries the *semantics*, not a literal API: one readable chain,
a facade per system, generated data. Each language reference adapts the
syntax to what its language expresses cleanly.

## Layers

Four roles, each calling **only the role directly below it**. The role names
here are descriptive, not prescriptive — projects call them plumbing/harness,
primitives/drivers, abstractions/actions, tests/specs; pick names that fit
the project. What matters is the dependency direction and what lives where.

| Role                 | Holds                                                                                                                                       |
| -------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| **scenarios**        | the tests: Given/When/Then chains only — no selectors, SQL, HTTP, or sleeps                                                                 |
| **business actions** | the facades scenarios speak through (`ui`, `db`, `mocks`): `ui.Login(customer)` clusters the clicks, `db.Customers.Add(customer)` inserts the graph |
| **system access**    | one technical wrapper per external system: db client, Playwright driver, mocked-service client, test-data generators                       |
| **harness**          | the Given/When/Then builder, scenario lifecycle (browser, db, mocks), per-step failure reporting                                            |

`ui`, `db`, and `mocks` are entry points, not god objects. Keep each root a
thin table of contents: a few cross-cutting actions (`ui.Login`) plus one
property per domain area — screen/flow objects under `ui` (`ui.Checkout.Pay(…)`,
the page-object pattern behind a facade), entity or aggregate facades under
`db` (`db.Orders`), one stub per external service under `mocks`. Grow by
adding areas, not root methods. Keep the three roots themselves, though —
they carry the scenario's meaning: `ui` is the front door, `db`/`mocks` the
back door.

A scenario seeds state through `db`/`mocks` (Given), acts through `ui` (When),
and asserts through any facade (Then). Seed through the database, not the UI —
UI setup is slow and brittle; the UI is for the behavior under test.

`Then` accepts a plain predicate or an assertion written with the project's
assertion library (FluentAssertions, AssertJ, `expect`, …) — the harness only
assumes *returns false or throws = not true yet* and polls either until a
timeout, because When steps trigger asynchronous backend work. Prefer the
assertion form when expected-vs-actual detail matters in the failure message.

## Workflow

Find the e2e folder (default `e2e/` at the repo root; follow the repo's test
folder convention if it has one).

- **Harness exists** → read the e2e folder's `CLAUDE.md`, then write tests.
- **No harness yet** → run Setup first.
- **Existing tests but no layers** → run Setup, port the most central existing
  test as the example slice, migrate the rest as they're touched.

### Writing a test

1. Generate fresh test data (`Customer.Generate()`) — never hardcode values;
   parallel tests share the database.
2. Express the scenario with existing facade actions.
3. Missing business action → add it to a facade, composed from system access.
   Missing technical capability → add a system-access wrapper. Never inline
   either into the test.

### Setup

1. Detect language and stack (test framework, db access, how the app under
   test runs); ask the user what isn't evident. Read the matching reference —
   [CSharp.md](CSharp.md), [Java.md](Java.md), or
   [TypeScript.md](TypeScript.md); for another language, adapt the nearest
   one.
2. Create the e2e project: a folder per layer and the harness from the
   reference.
3. Cut one thin slice through all layers for the project's most central flow:
   a generator, the system access, a few facade actions, **one example test**.
4. Write the e2e `CLAUDE.md` from the template below, with the real folder
   names and the example test.
5. Build and run the example test; fix until it passes — or until only the
   app-under-test environment is missing, then tell the user what to start.

## e2e CLAUDE.md template

```markdown
# E2E Tests

BDD scenarios written directly in code: tests read like business scenarios;
all technical detail lives in lower layers. Each layer calls only the layer
directly below it.

| Role             | Folder     | Holds                                                  |
| ---------------- | ---------- | ------------------------------------------------------ |
| scenarios        | `<folder>` | Given/When/Then chains — no selectors, SQL, HTTP, sleeps |
| business actions | `<folder>` | `ui`/`db`/`mocks` facades built from system access     |
| system access    | `<folder>` | db client, Playwright driver, mock client, generators  |
| harness          | `<folder>` | Given/When/Then builder + lifecycle — rarely changes   |

## Writing a test

1. Generate fresh data (`Customer.Generate()`) — never hardcode; tests share
   the database.
2. Express the scenario with existing facade actions.
3. Missing action → add it to the matching area facade (screen/flow under
   `ui`, entity under `db`). Missing technical capability → add a
   system-access wrapper. Typing a selector or SQL inside a test means a
   layer is missing — stop and add it below instead.
4. `Then` takes a predicate or an assertion (any assertion library); both are
   polled until a timeout — never add manual waits.

## Example

<the example test>

## Running

<how to start the app under test and run the suite>
```
