# E2E in TypeScript

## Stack

Follow what the repo already uses; these are the defaults when there is no signal.

| Concern         | Default                                                                  |
| --------------- | ------------------------------------------------------------------------ |
| Test runner     | `@playwright/test` — fixtures, parallel workers, traces, retries built in |
| Mocked services | `mockttp` — a real HTTP server the app's outbound calls point to          |
| Generators      | hand-rolled `generate()` factories (`@faker-js/faker` if data is rich)    |
| DB access       | the app's own client if importable (Prisma/Drizzle/knex); otherwise `pg`  |

Install browsers once: `npx playwright install chromium`. Let the runner do
the operational work — `screenshot: 'only-on-failure'` and
`trace: 'retain-on-failure'` in the config replace hand-rolled failure
capture; `workers` and per-test isolation come free.

## Layout

Folder names follow the layer roles here — rename to fit the project.

```text
e2e/
├── CLAUDE.md
├── package.json
├── playwright.config.ts   baseURL from E2E_BASE_URL, screenshots/trace on failure
├── harness/
│   ├── scenario.ts        Given/When/Then builder (below)
│   └── fixtures.ts        extends the base test with the scenario fixture
├── system/
│   ├── database.ts        pg pool / app's db client
│   ├── mock-server.ts     mockttp wiring
│   └── generators.ts      Customer.generate(), Order.generate(), …
├── actions/
│   ├── db.ts              db.customers, db.orders, …
│   ├── ui.ts              ui.login, ui.placeOrder, ui.cart, …
│   └── mocks.ts           mocks.paymentProvider, …
└── scenarios/
    └── order.spec.ts
```

## Harness — `scenario.ts`

This exact shape is verified to compile (strict) and run; keep it. Two
TypeScript-specific constraints shape it — don't copy the C#/Java shape:

- **One facades object, destructured per step** (`({ db }) => …`) instead of
  per-facade overloads — TypeScript picks overloads in declaration order
  without checking whether the lambda body binds, so C#-style resolution is
  unreliable. Destructuring is the idiom Playwright fixtures already use.
- **`then` is dual-mode** because any object with a `then` method is a JS
  *thenable*: `await chain` calls `then(onFulfilled, onRejected)` with two
  functions. One function argument = DSL assertion; a function second argument
  = the await protocol, which runs the recorded steps. That keeps the chain
  awaitable with no terminal `.run()` to forget.

One `then` serves two styles with one polling rule — retry while the step
returns `false` (predicate) or throws (assertion); anything else passes.
Playwright's `expect` (or any assertion library) works as-is, and its
expected-vs-actual message survives into the timeout failure.

```typescript
import type { Db } from '../actions/db';
import type { Ui } from '../actions/ui';
import type { Mocks } from '../actions/mocks';

export type Facades = { db: Db; ui: Ui; mocks: Mocks };
type Step = (f: Facades) => void | Promise<void>;
type Check = (f: Facades) => boolean | void | Promise<boolean | void>;

const ASSERT_TIMEOUT_MS = 10_000;

export class Scenario {
  constructor(private facades: Facades) {}
  private steps: Array<{ keyword: string; text: string; run: (f: Facades) => Promise<void> }> = [];

  given(step: Step): this { return this.add('Given', step); }
  when(step: Step): this { return this.add('When', step); }

  then(check: Check): this;
  then<T>(onFulfilled: (value: void) => T | PromiseLike<T>, onRejected: (reason: unknown) => unknown): Promise<T>;
  then(arg: Check | ((value: void) => unknown), onRejected?: (reason: unknown) => unknown): unknown {
    if (typeof onRejected === 'function') {
      // `await scenario` / Promise.resolve(scenario) lands here
      return this.run().then(arg as (value: void) => unknown, onRejected);
    }
    const check = arg as Check;
    const text = check.toString();
    this.steps.push({
      keyword: 'Then', text,
      // when-steps trigger async backend work; asserting on the first read races it.
      // retry while the check returns false (predicate) or throws (assertion).
      run: async (f) => {
        const deadline = Date.now() + ASSERT_TIMEOUT_MS;
        for (;;) {
          try {
            if ((await check(f)) !== false) return;
            if (Date.now() > deadline)
              throw new ScenarioFailedError(`Then ${text} — still false after ${ASSERT_TIMEOUT_MS / 1000}s`);
          } catch (e) {
            if (e instanceof ScenarioFailedError) throw e;
            if (Date.now() > deadline)
              throw new ScenarioFailedError(`Then ${text} — still failing after ${ASSERT_TIMEOUT_MS / 1000}s: ${(e as Error).message}`, e);
          }
          await new Promise((resolve) => setTimeout(resolve, 250));
        }
      },
    });
    return this;
  }

  private add(keyword: string, step: Step): this {
    // step.toString() carries the lambda source into failure messages
    this.steps.push({ keyword, text: step.toString(), run: async (f) => { await step(f); } });
    return this;
  }

  private async run(): Promise<void> {
    for (const { keyword, text, run } of this.steps) {
      try { await run(this.facades); }
      catch (e) {
        if (e instanceof ScenarioFailedError) throw e;
        throw new ScenarioFailedError(`${keyword} ${text} — threw ${(e as Error).name}: ${(e as Error).message}`, e);
      }
    }
  }
}

export class ScenarioFailedError extends Error {
  constructor(message: string, public override cause?: unknown) { super(message); }
}
```

## Harness — `fixtures.ts`

```typescript
import { test as base } from '@playwright/test';
import { Scenario } from './scenario';
import { Db } from '../actions/db';
import { Ui } from '../actions/ui';
import { Mocks } from '../actions/mocks';
import { pool } from '../system/database';
import { mockServer } from '../system/mock-server';

export const test = base.extend<{ scenario: () => Scenario }>({
  scenario: async ({ page }, use) => {
    await use(() => new Scenario({
      db: new Db(pool),
      ui: new Ui(page),
      mocks: new Mocks(mockServer),
    }));
  },
});
export { expect } from '@playwright/test';
```

Start the mock server once in `globalSetup` on a fixed port the app's
outbound config points to; worker processes control it through mockttp's
admin API (`getAdminServer()` in setup, `getRemote()` in `mock-server.ts`) —
workers are separate processes and can't share the live server object.

How the app under test starts (docker compose, `npm run dev`, deployed URL)
is project-specific — ask the user, wire it in (Playwright's `webServer`
config option can own it), and record it under **Running** in the e2e
`CLAUDE.md`.

## Actions, system access, generators (skeleton)

Facade roots stay thin: one property per domain area (screens/flows under
`Ui`, entities under `Db`), actions on the area objects — grow by adding
areas, not root methods.

```typescript
// actions/ui.ts — clusters Playwright primitives into business actions
import type { Page } from '@playwright/test';

export class Ui {
  constructor(private page: Page) {}
  get cart() { return new Cart(this.page); }

  async login(customer: Customer) {
    await this.page.goto('/login');
    await this.page.getByLabel('Email').fill(customer.email);
    await this.page.getByLabel('Password').fill(customer.password);
    await this.page.getByRole('button', { name: 'Log in' }).click();
  }
}

// actions/db.ts — inserts whole entity graphs, queries for assertions
export class Orders {
  constructor(private pool: Pool) {}

  async contains(order: Order, customer: Customer): Promise<boolean> {
    // fresh query per call — then() polls this
    const { rows } = await this.pool.query(
      `select count(*)::int as n from orders o
         join customers c on c.id = o.customer_id
        where c.email = $1 and o.sku = $2`,
      [customer.email, order.sku],
    );
    return rows[0].n > 0;
  }
}

// system/generators.ts — e2e-side test data, not the app's entities
export type Customer = { email: string; password: string; name: string };
export const Customer = {
  generate(): Customer {
    const id = crypto.randomUUID().slice(0, 8);
    return { email: `e2e-${id}@example.test`, password: crypto.randomUUID(), name: `E2E Customer ${id}` };
  },
};
```

## Example test

```typescript
import { test, expect } from '../harness/fixtures';
import { Customer, Order } from '../system/generators';

test('customer can order and pay', async ({ scenario }) => {
  const customer = Customer.generate();
  const order = Order.generate();

  await scenario()
    .given(({ mocks }) => mocks.paymentProvider.acceptsAll())
    .given(({ db }) => db.customers.add(customer))
    .when(({ ui }) => ui.login(customer))
    .when(({ ui }) => ui.placeOrder(order))
    .then(({ db }) => db.orders.contains(order, customer))
    .when(({ ui }) => ui.checkoutAndPay(customer))
    .then(({ ui }) => ui.cart.isEmpty())
    .then(async ({ db }) => expect(await db.orders.statusOf(order, customer)).toBe('Ordered'));
});
```

The first two `then`s are predicates; the last is an assertion — prefer the
assertion form when expected-vs-actual matters in the failure message.

Playwright runs spec files in parallel workers — generated data is what keeps
them from colliding in the shared database.
