# 09 — Web front end

A single-page application that talks to the API through the generated client and to
nothing else.

---

## 1. Structure

```
web/src/
  api/
    client.generated      generated client — never hand-edited
    types.generated       generated contract types — never hand-edited
    core/                 transport: credential, headers, errors, timeout
    <feature>Api.ts       hand-written wrappers for binary/multipart only
  pages/                  route-level components, one per route
  components/             shared components, and per-feature subfolders
  <domain>/               feature modules: state, hooks, and domain helpers
  hooks/
  layouts/
  config/                 runtime configuration read once
  utils/
  test/                   setup, fixtures
```

Rules:

- **No top-level `shared/` dump.** Shared things live in a folder named for what they
  are.
- **A component file holds one component.** A file that exports five components is a
  folder.
- **Routes are lazy-loaded per feature.** The initial bundle carries the shell and the
  first screen.
- **Route guards live with the auth module**, not inline in each route.

---

## 2. Calling the API

```ts
const { orders, nextCursor } = await apiClient.orders.list({ customerId, limit: 50 });
```

Rules:

1. **Never hand-declare a type that duplicates a generated one.** It stops tracking the
   contract the moment the contract changes, silently.
2. **Never call the transport directly from a component.** Components call a hook or a
   service; the hook or service calls the client.
3. **Every call has a timeout.** Set it in the transport core, once.
4. **Handle every error branch explicitly**, with a specific, human-readable message.
   Banned: a catch-all "Something went wrong" that collapses distinct failures; an error
   branch left unhandled; a raw error code shipped as user copy. When the server adds a
   new error value, every consumer gains a matching real message.

---

## 3. State and data loading

Pick one model for the whole application and hold to it. Two that work:

**A — hooks plus a small cache layer** (React-shaped). A per-domain hook owns the read,
the loading state, and the error state, and exposes a refresh function. Mutations call
the client and then refresh the collection they affected.

**B — services plus reactive values** (Angular-shaped, reference implementation B). A
per-domain service constructs its caches in its constructor, exposes a reactive value per
collection, and subscribes to the mutation use cases to refresh the right cache:

```ts
merge(create.onExecute, update.onExecute, remove.onExecute)
  .subscribe(() => this.orderCache.refresh());
```

Whichever you pick:

- **Components consume the domain layer, never the API client.**
- **Caches are constructed eagerly**, in the constructor or at module scope — never
  lazily on first call, which races.
- **Every cache has a defined empty value** so a consumer never has to handle
  `undefined`. A collection cache's empty value is an empty collection that still
  answers `findById` with a sentinel rather than nothing.
- **Derived state is computed, not recomputed by hand** in a lifecycle hook.

### 3.1 A write the user cannot see is a failed write

After a create, delete, rename, or move succeeds, the handler **must** either re-read the
collection the item belongs to or update the bound state in place. Doing neither is the
bug, and it reads to the user as a failure — so they retry, and now there are two.

- Refreshing a *sibling* collection is not enough. Refresh the one that receives the
  item.
- If the new item lands inside a collapsed or paginated container, reveal it, or the
  refresh is invisible anyway.
- The end-to-end test asserts the item is **rendered**, by its test id — not that the
  call resolved. Without that assertion the rule is unenforced, because the obvious test
  passes on the half that worked.

### 3.2 No polling

There is a push channel for this. A poll added because push "seemed flaky" hides the
broken push path and the breakage then sits there indefinitely. Lengthening the interval
does not make it acceptable.

- No timer-driven re-fetch of server state. A timer driving *local* state — a clock, a
  recording duration, a countdown — is fine and is not what this bans.
- Every reconnect and every error-triggered retry has a **maximum attempt count and a
  growing delay**. On exhaustion, surface it and report it. Giving up silently is a
  swallow.

---

## 4. Forms and inputs

- A form's shape is declared in one place, not spread across the template.
- Branch logic, data manipulation, and validation live in the component or its model —
  never in the template.
- Validation messages are specific and actionable. They name the field and what is wrong
  with it.
- Where the product has configurable terminology, **no user-facing string hardcodes a
  configurable word.** Route it through the terminology layer. A hardcoded literal
  renders the wrong word for the tenant that renamed it.

---

## 5. A control that writes disables itself while the write is out

Any control that fires a server write on click — save, delete, send, publish, approve,
retry, pay, generate — holds a loading flag for the duration of the call and binds it.

- Set the flag **before** the await; clear it in a `finally`. Clearing it after the call
  leaves the control dead for good if the call throws.
- Guard re-entry **at the top of the handler**, not only on the binding. Keyboard
  activation and a click landing before the re-render both reach the handler without
  passing through the disabled state.
- For a list where each row writes independently, key the flag per row so one busy row
  does not lock the others.

**Why:** a spinner on its own stops nobody. The control stays clickable for the whole
round trip, so a second click sends the write twice — two invitation emails with the
first link now dead, a duplicate charge, the same request approved twice.

A flag that lives in a service is not a guard on its own. The service knowing it is busy
does nothing unless the handler checks it and the control binds it.

---

## 6. Loading, empty, and error states

Every screen that reads data renders four states, and all four exist from the first
version:

| State | Requirement |
|---|---|
| Loading | Visible. A spinner or skeleton, not a blank region. |
| Loaded, empty | Says it is empty and what to do about it. |
| Loaded, populated | The normal case. |
| Failed | Visibly failed. **Never drawn as empty.** |

**A lifecycle hook starts work and returns; the component holds the state and the
template draws it.** Never await inside a lifecycle hook. A hook awaiting something that
never settles never returns, the component stays un-loaded, and that state usually looks
finished — an empty list reads as "there is nothing here" — while throwing nothing and
logging nothing, so no gate can see it.

A bounded timeout is not the fix: it converts a hang into an error the reader did not ask
about and cannot act on. A spinner is the correct answer whether the read takes 200ms or
never returns.

**Testing it:** assert the hook is *done* when it returns, not merely that it was called.
Adopt what the hook returns and race it. Calling the hook and ignoring its return value
passes whether or not it awaits, which is a vacuous test.

---

## 7. Design system

- **One in-repo component set is the default** for anything a screen renders itself:
  buttons, cards, banners, tags, status pills, chips, spinners, skeletons, dialogs,
  form fields. Build it early, in one folder, with a gallery route that renders every
  component and every one of its states.
- **Reach for a third-party widget only where the set has no equivalent** — a data grid,
  a calendar popup, a map, a chart, a scheduler. List those exceptions in the
  architecture guide.
- **Adding a new component to the set is an architectural decision**, because every
  screen inherits its API. So is adding a new UI dependency.
- **Icons come from one set.** A missing glyph in some icon systems renders as literal
  text with no warning, so look at the rendered screen, not just the markup.
- **Colour comes from tokens.** No new hardcoded colour in a component stylesheet, and no
  inline colour. Theming is central.
- **Watch third-party component theming.** A library that exposes "system" tokens usually
  bakes its *component* tokens from its own palette, which then renders the library's
  colours beside yours. When you introduce such a component: look at it rendered, and if
  it is off-palette, repoint its component tokens centrally — never per screen, never
  with a hardcoded colour.

---

## 8. Test identifiers

Every element a test interacts with carries a stable test identifier attribute.

- Pattern: `{feature}-{component}-{element}[-{dynamicId}]`, kebab-case.
- Adding one is part of writing the component, not part of writing the test.
- A build-time check can enumerate them and fail on duplicates or on a screen with none.

This is what makes the end-to-end suite readable and stable. See [13-testing-e2e](13-testing-e2e.md).

---

## 9. Component tests

Two kinds, with different jobs:

**Logic tests** — construct the component's model or hook directly and test its methods.
No rendering. Fast. This is where branching, validation, and state transitions are
proven.

**Rendering tests** — mount the component in a DOM environment and assert what it draws:
content projection, what a conditional hides, what the empty state says, which state a
failed read renders. Milliseconds, and they answer questions the logic test cannot.

Both mock the API at the client boundary. **Never hit the network in a unit test.**

A rendering test is not a substitute for an end-to-end test, which is the only thing
proving a real route against a real backend.

### 9.1 Mind the environment gap

If a screen is gated on runtime configuration — a feature flag, an API key, a
capability — then in a test environment where that configuration is absent the screen
never mounts and the test proves nothing. Mock the configuration module in the component
test, and make sure CI supplies whatever the build needs.

---

## 10. Errors and observability in the client

- A global error boundary catches render failures and reports them, with a screen that
  offers a way forward.
- `console.*` is not observability. It is invisible in production and it is a swallow by
  another name. Route errors through the reporting helper.
- Report unhandled rejections and resource-load failures too.
- Never put personal data or a credential into an error report's tags or URL.

---

## 11. Layout assertions

If you assert on layout — that a menu does not overflow, that a panel fits — measure
against the element's **offset parent**, not the viewport, and not the document. Inside a
horizontally scrolling container both of those measure the container: a clipped element
reads as on-screen and the document width reflects the scrollable region.

And prove the assertion bites: remove the fix and watch it go red. A layout assertion
that has never failed has not been shown to measure anything.
