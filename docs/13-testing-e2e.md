# 13 — End-to-end testing

The end-to-end suite drives the real user interface against a real backend. It is the
only thing in the estate that proves a real route, a real contract, and a real user
journey.

Copy this file into the new repository as `web/tests/e2e/GUIDE.md`.

For checking an existing suite against these rules rather than writing to them, the
`e2e-coverage-validation` skill in
**[paulmorrishill/ClaudeAnnoyanceReduction](https://github.com/paulmorrishill/ClaudeAnnoyanceReduction)**
audits coverage and reports the gaps.

---

## 1. Hard rules

### 1.1 No faking API responses

**Never intercept a request and return a success payload the real backend could have
produced from seeded data.** Seed the data and let the backend answer. A faked response
decouples the test from the thing it exists to exercise: the assertion then proves the
fixture, and the endpoint can break with the test still green.

Request interception itself is not banned. Three uses are allowed:

**a. Delaying a real response.** A handler that awaits and then continues the request is
not a mock: it produces no body and changes no status, so the real request reaches the
real backend. Only the timing changes. Use it when the case is about a window — a busy
state, a race between a save and a reload — that would otherwise close between two
assertions on a fast machine. Widening a timeout instead makes the case unable to fail
for the reason it exists.

> **Count the handler.** A pattern that silently stops matching — a renamed endpoint, an
> added path prefix, a glob that no longer covers the URL — leaves the handler installed
> and never called, so the delay does nothing and the case races again with every run
> still green. Increment a counter inside the handler and assert it after release.

**b. External services that are non-deterministic or cost money.** Model calls, payment
providers, third-party lookups.

**c. Forcing an error the backend cannot be asked for on demand**, when the assertion is
about how the interface handles the failure rather than about the data. A 500 from a
mutation, a 502 from a token endpoint. Note that generic server-error handling does not
*need* an end-to-end case; where one exists, this is how it is written.

### 1.2 Select elements only by test identifier

Never by CSS class, tag, DOM structure, or visible text. Every element a test touches
carries a test id. If the component lacks one, add it — that is part of writing the
component.

### 1.3 All interaction goes through page objects

Tests never reach for an element directly for anything used more than once. Wrap it in a
page object.

### 1.4 Seed through the API or the interface

Either an API helper that calls real endpoints, or drive the interface to create the
data. **Never insert directly into the store.** A direct insert can write a row shape the
application could never produce.

### 1.5 Every test is independent

A reset before each test wipes all state. No test depends on data another test created.

---

## 2. Layout

```
web/tests/e2e/
  support/
    local-environment.ts    reset, authentication, mail assertions
  helpers/
    api.helper.ts           seeding functions built on real endpoints
    <feature>.helper.ts     feature-specific seeding
  pages/
    order.page.ts           page objects, one file per feature area
  fixtures/                 static files used by upload tests
  order-flows.spec.ts       specs, one file per feature area
  GUIDE.md                  this file
```

---

## 3. Page objects

One file per feature area, one class per distinct screen.

```ts
export class OrderListPage {
  constructor(private readonly page: Page) {}

  async goto(): Promise<void> {
    await this.page.goto('/orders');
  }

  get title()        { return this.page.getByTestId('order-list-title'); }
  get createButton() { return this.page.getByTestId('order-list-create-button'); }

  row(orderId: string) {
    return this.page.getByTestId(`order-list-row-${orderId}`);
  }

  async createOrder(customerName: string): Promise<string> {
    await this.createButton.click();
    await this.page.getByTestId('order-create-customer-input').fill(customerName);
    await this.page.getByTestId('order-create-submit-button').click();
    return await this.page.getByTestId('order-detail-id').innerText();
  }
}
```

Conventions:

- Constructor takes the page and stores it privately.
- Static elements are accessors returning a locator.
- Elements parameterised by an identifier are methods.
- A multi-step flow may be a method that returns what the test needs next.
- **A two-state control gets both an opener and a closer.** See § 7.

---

## 4. Test identifiers

Pattern: `{feature}-{component}-{element}[-{dynamicId}]`, kebab-case.

```tsx
<h1 data-testid="order-detail-title">{order.reference}</h1>
<button data-testid="order-detail-cancel-button">Cancel</button>
<li data-testid={`order-list-row-${order.orderId}`}>…</li>
```

A build-time script can enumerate them and fail on duplicates.

---

## 5. The lifecycle

Every spec follows the same five steps:

1. **Reset** — wipe the store, the authentication state, the mail simulator, the file
   store.
2. **Authenticate** — through the real sign-in interface, as a named role.
3. **Seed** — create prerequisites through API helpers, or through the interface when the
   creation flow is what is under test.
4. **Act** — through page objects.
5. **Assert** — on rendered state, by test id.

```ts
test.beforeEach(async ({ request, page }) => {
  await resetLocalEnvironment(request);
  await page.context().clearCookies();
});

test('an order can be cancelled', async ({ page }) => {
  await signInAs(page, 'editor');

  const order = await seedOrder(page, { customerName: 'Acme' });

  const detail = new OrderDetailPage(page);
  await detail.goto(order.orderId);
  await detail.cancelButton.click();
  await detail.confirmButton.click();

  await expect(detail.status).toHaveText('Cancelled');
});
```

### 5.1 Authenticate through the real sign-in interface

Do not write a credential into browser storage to skip the sign-in screen. A run that
starts after the sign-in screen cannot find a defect in the sign-in screen — and sign-in
is the one flow every user takes.

### 5.2 Seeding helpers call real endpoints

```ts
export async function seedOrder(page: Page, opts: { customerName: string }) {
  return (await apiPost(page, '/orders/create', opts)) as { orderId: string };
}
```

`apiPost` runs inside the browser context so it picks up the real credential
automatically.

---

## 6. What every spec must assert

- **A create asserts the new item is rendered**, by its test id — not that the call
  resolved. This is the assertion that catches "the write succeeded and the user cannot
  see it" ([09-frontend](09-frontend.md) § 3.1). Without it the rule is unenforced.
- **A delete asserts the row is gone from the DOM**, not that the delete resolved.
- **A permission-gated surface is asserted both ways**: visible and usable for a
  permitted role, absent or refused for an unpermitted one.
- **A tenant-scoped feature is asserted for leakage**: created in tenant A, absent in
  tenant B's list, search, and client sync payload.

---

## 7. Two-state controls

A test that opens a collapsible section proves it opens. It says nothing about closing,
and a regression leaving every section permanently open passes the whole suite.

- **Cover both directions**, and **enumerate the instances** rather than testing one. A
  per-instance table makes a new uncovered instance a missing row.
- **Assert the second state after the render that follows it.** An absence assertion
  passes at the first moment the element is missing, so a control that the next render
  puts back satisfies a single check. Read the state again after a wait, and again after
  provoking an unrelated re-render — a value re-derived on every render is exposed by the
  second read, never the first.
- **A blind click on a control whose state you have not established is a toggle, not an
  open.** It can pass only because the toggle did nothing.
- Related design rule: a route-based default belongs in a state *change*, not in the
  rendered expression. An expression like `isOpen || isOnThisRoute` cannot be closed while
  the condition holds.

---

## 8. Screens with volume

A screen backed by a growing collection gets a spec seeded with **hundreds of rows**,
asserting the screen stays correct and that its pagination controls work — forward, back,
and page size. A layout is correct on a five-row seed and falls over on real data.

Where the project keeps a screenshot catalogue (§ 11), that catalogue includes the
large-data state.

---

## 9. Running and reading

```
e2e                 whole suite
e2e <file>          one spec
e2e -g "<name>"     one test
e2e --headed        with a visible browser
```

Rules:

- **Prove a new or changed spec by running it more than once with retries disabled.**
  Three runs is the working standard. A spec that passes once has not been shown to be
  stable.
- **A retry-pass is a defect to triage, not a pass.** Read first-attempt failures
  separately from the final counts.
- **Never reuse a running server.** If the harness finds the port occupied, stop and ask
  — do not attach to whatever is there. A suite run against a stale build measures the
  old code.
- **Never run the suite on a machine that is also a CI runner** for the same suite. The
  two collide and the CI result is destroyed.
- The full suite runs after a block of merges on the integration branch, not once per
  pull request. Each pull request proves its own specs; the full run is the integration
  question, worth asking once per block.

---

## 10. Mobile-web and device suites

Where the mobile application also runs in a browser, a second configuration can run a
subset of specs against it. Keep the configurations in agreement with a parity test that
fails when one gains a setting the other lacks.

Device flows for the native app are a separate tool and a separate job. They cover
sign-in, the main task, and offline behaviour — not every screen.

---

## 11. Screenshot catalogue

Optional, and valuable: a job that drives each screen into each of its distinct states
and captures an image — populated, **empty**, **loading**, **failed**, and each
**role** variant, plus the large-data state from § 8.

A new screen, or a new state on an existing screen, is not done until its capture exists.
States that are never captured are never seen, and the empty, failed, and overflowing
views are exactly where layout defects hide.
