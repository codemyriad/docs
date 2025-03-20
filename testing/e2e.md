## purpose

A reference guide for writing end-to-end (e2e) tests with Playwright.

This document serves two main purposes:

- Ensuring tests remain _readable_ and _stable_
- Documenting _solutions for scenarios_ where no immediately obvious approach was available

It is a _living document_, meant to evolve as we refine our practices. Updates should be made as we learn, and existing patterns should be challenged when they no longer prove effective.

### test setup

#### test data

1. Test data should be consistent. In most cases, test data should remain the same across all test runs. For example, `book1` should always reference the same `isbn`, `title`, `authors`, etc. This consistency makes debugging easier and helps maintain sanity. It is especially crucial when testing scenarios at the end of a series of user flows, where test data accumulates into a final result that we need to assert against.
2. Test data should be easy to reference. Store all test data in a single file as clearly defined declarations. Ideally, it should be [annotated with JSDoc comments](https://github.com/librocco/librocco/blob/main/apps/e2e/helpers/fixtures.ts#L97C1-L171C3) to improve visibility across other files (e.g., via IDE tooltips).
3. While consistency is important, we recognize that we should occaisonally introduce randomness:
   1. To generate consistent test data upfront. This helps us avoid our tendencies to use the same unimaginative strings ("janedoe@mail.com") in every project. FakerJs' guide on [creating complex objects](https://fakerjs.dev/guide/usage.html#create-complex-objects) is useful for setting up a structured “seed” script.
   2. For test cases that explicitly aim to check how a system handles variability.

#### fixtures

##### notes

The [playwright docs](https://playwright.dev/docs/test-fixtures) provide a comprehensive intro. The key highlights are:

1. Fixtures establish an isolated environment for each test. They are useful for:
   - setting up/tearing down state that needs to run before/after any number of similar cases
   - initialising & sharing resources-utilities with a test environment
2. There are some built-ins: `page`, `browser`, `context`, `request`. When these are referenced in a test function, we use the defaults that Playwright provides us. But _they can also be overridden_. For instance, the following overrides the `page` fixture by automatically navigating to a `baseURL`:

   ```typescript
   import { test as base } from "@playwright/test";

   export const testNav = base.extend({
     page: async ({ baseURL, page }, use) => {
       await page.goto(baseURL);
       await use(page);
     },
   });
   ```

   - This means that any case which uses `testNav("navigation should do...", async ({ page }) => {})` will navigate _before_ any of the actions or assertions that are written in the test.

3. We can [create our own custom fixtures](https://playwright.dev/docs/test-fixtures#creating-a-fixture)
4. [They have advantages over `before`/`after` hooks](https://playwright.dev/docs/test-fixtures#with-fixtures)
5. By default, they are "scoped" to each "test run", meaning they are setup for each test case.
   - But they can also be [scoped to a "test worker"](https://playwright.dev/docs/test-fixtures#overriding-fixtures). What does this mean? Playwright uses [worker processes](https://playwright.dev/docs/test-parallel) to run test files. It will reuse each process for as many test files as it can, and where environments match, it can setup the fixture once and reuse it each time.
6. They can be [run automatically](https://playwright.dev/docs/test-fixtures#overriding-fixtures)
7. They can be [made optional](https://playwright.dev/docs/test-fixtures#overriding-fixtures) , which means they can be defined as a parameter per [project](https://playwright.dev/docs/test-projects) (see [projects](#projects))
8. [Execution Order](https://playwright.dev/docs/test-fixtures#overriding-fixtures) matters when you consider the 3 points above.
9. They can [be merged](https://playwright.dev/docs/test-fixtures#overriding-fixtures)
10. _They are invoked by reference._ This is an important point on their use. If we don't reference the fixture in the test case, it won't run its setup. This is true for any built-in fixtures too.

    - As a result of this, fixtures can be chained: if we reference a fixture as a parameter to another fixture, it will run before the latter. e.g:

      ```typescript
      import { test as base } from '@playwright/test';

      export const testNav = base.extend({
        user: async({}, use) => {
      	  const user = await do(signup or signin)
      	  use(user)
        },
        page: async ({ user, page }, use) => {
          await page.goto(`/account/${user.id}`);
          await use(page);
        },
      });
      ```

##### rules

1. Fixtures should have a single responsibility. Each fixture should set up only one piece of state or a single utility-resource. Instead of handling multiple concerns, fixtures should be chained together. For example, in an SQL model with two related tables—`customer_orders` and `customer_order_lines`—each should be set up as an independent fixture, with `customer_order_lines` referencing `customer_orders`.
2. Fixtures should return the test data declaration, not the result of a database call. Instead of returning data retrieved from the database, fixtures should provide the test data declaration —the values that are being inserted or used. This ensures tests remain predictable and do not depend on the database.

   ```typescript
   const customer_orders = [];

   export const testNav = base.extend({
     customerOrders: async ({}, use) => {
       await upsertCustomerOrder(customer_orders);

       use(customer_orders);
     },
   });
   ```

##### examples

1. [This repo](https://github.com/ncrmro/remix-supabase-playwright/blob/main/e2e/fixtures.ts#L10) acted as some initial inspiration:
   - It sets up a Supabase client as a worker fixture
   - It adds a `user` test fixture which creates an authenticated session. This allows test cases which use this fixture to bypass any login steps. It does this by: creating a new user in the db, signing them in with the Supabase client and emulating a session by adding the Supabase cookies to the browser context (see [working with browser state: cookies](#cookies)).
2. [WhatIsEdible's fixtures](https://github.com/whatisedible/Edible/blob/main/apps/e2e/lib/fixtures.ts):
   - It overrides the `page` fixture to check for a hydration flag after any navigation event. This makes sure that the page is fully interactive before any assertions are made. (see [page hydration](#page-hydration))
   - It instantiates our i18n lib, so that tests can use our dictionary strings when selecting elements or when asserting against text content in the UI. We could also pass through the `locale` from the project setup, so that we could easily run the same specs for different langs.
   - It composes fixtures effectively.
     - `testBase` sets up shared utilities required for all test cases. This is used for **unauthenticated** scenarios, such as asserting against public parts of the app.
     - `testAuthenticated` extends `testBase`, adding setup for a `user` and `account`, overriding `page` to sign the user in (see [working with browser state: cookies](#cookies)), and introducing additional fixtures for domain entities like `restaurant` and `menu`.
3. [Librocco's fixtures](https://github.com/librocco/librocco/blob/main/apps/e2e/helpers/fixtures.ts):
   1. It annotates fixture data with JSDoc comments so that the data can be referenced by tooltip in other files.
   2. It sets up highly modular fixtures, ensuring that each test depends only on the specific domain entities it requires.
   3. It uses [Playwright's `evaluateHandle`](https://playwright.dev/docs/api/class-page#page-evaluate-handle) to write to the test page's browser storage. This allows setup to be run in apps that depend on local storage (see [working with browser state: indexedDb](#indexeddb))

#### page hydration

Waiting for page hydration is essential to ensure that SvelteKit (or any modern framework) has finished loading before Playwright begins making assertions. Skipping this step can lead to flaky tests that fail unpredictably without a clear cause.

[This blog post](https://spin.atomicobject.com/hydration-sveltekit-tests/) was a lifesaver after much frustration. It references [this Playwright issue](https://github.com/microsoft/playwright/issues/19858#issuecomment-1377088645), which points to the Playwright setup in the official Svelte repo—so you know it’s solid.

#### working with browser state

##### indexedDb

> :construction: TODO: Improve Write Up

When an application relies on local storage, setting up test data can be challenging. This is because each test page’s browser context operates in isolation from Playwright’s workers. [@ikutsu](https://github.com/ikusteu) has implemented a solution for managing browser storage during test setup:

- [`getDbHandle`](https://github.com/librocco/librocco/blob/main/apps/e2e/helpers/db.ts) wraps [Playwright’s `evaluateHandle`](https://playwright.dev/docs/api/class-page#page-evaluate-handle). A [JSHandle](https://playwright.dev/docs/api/class-jshandle) represents an in-page JavaScript object, acting as a bridge between different execution contexts.
- This handle is used to evaluate database handlers in [our fixtures](https://github.com/librocco/librocco/blob/main/apps/e2e/helpers/fixtures.ts), enabling test setup operations like upserting customers, books, and suppliers.
- Our handlers are exposed globally via `window` when `IS_E2E` is set [in the route layout](https://github.com/librocco/librocco/blob/main/apps/web-client/src/routes/%2Blayout.svelte)
- [Potential improvements and open discussions](https://github.com/librocco/librocco/issues/780)

##### cookies

To emulate a user session while using Supabase, the cookies returned from `signInWith...` need to be set in the test page’s context. Playwright provides [documentation on managing authenticated state](https://playwright.dev/docs/auth), but the recommended approach didn't work in this case. The solution currently used in WhatisEdible can be found in [this utility](https://github.com/whatisedible/Edible/blob/main/apps/e2e/lib/utils.ts#L148C1-L188C3).

#### supabase

1. [Testing Supabase Magic Login in CI with Playwright](https://www.bekapod.dev/articles/supabase-magic-login-testing-with-playwright/)

#### projects

> :construction: TODO: I am exploring them with Edible, for running the same tests with different user accounts (free vs premium subscriptions). I will report back - Chris

We currently don't use [projects](https://playwright.dev/docs/test-projects) for any reason other than running our tests in different browsers. They could be useful for more complex testing scenarios:

1. [Coordinating global setup and teardown](https://playwright.dev/docs/test-global-setup-teardown)
2. [Parameterizing projects](https://playwright.dev/docs/test-parameterize#parameterized-projects)
