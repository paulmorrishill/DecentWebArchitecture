# 11 — Unit testing

The use-case unit test is the dominant test in this architecture. It runs in
milliseconds, needs no container and no database, and is where almost all behaviour is
proven.

---

## 1. The pattern

Construct the use case with fake ports, drive it with a request context, assert on both
the returned value and the side effects the fake captured.

```ts
const baseContext: RequestContext = {
  requestId: 'req-1',
  userId: 'user-1',
  userEmail: 'user@example.com',
  userRoles: ['Editor'],
  tenantId: 'tenant-a',
  nowIso: '2026-01-01T00:00:00.000Z',
};

describe('CreateOrderUseCase', () => {
  it('creates a draft order with the priced lines', async () => {
    let created: OrderEntity | null = null;

    const orderRepository: OrderRepository = {
      async createOrder(record) { created = { ...record }; },
      async getOrderById() { return null; },
      async listOrdersForCustomer() { return []; },
      async cancelOrder() { return null; },
    };

    const useCase = new CreateOrderUseCase(orderRepository, stubPricing());

    const response = await useCase.execute(
      { customerId: 'cust-1', lines: [{ productId: 'p1', quantity: 2 }] },
      baseContext,
    );

    expect(created).not.toBeNull();
    expect(created!.status).toBe('draft');
    expect(created!.createdAt).toBe('2026-01-01T00:00:00.000Z');
    expect(response.order.lines[0]!.quantity).toBe(2);
  });

  it('rejects an order with no lines above zero quantity', async () => {
    const useCase = new CreateOrderUseCase(emptyRepository(), stubPricing());
    await expect(
      useCase.execute({ customerId: 'cust-1', lines: [{ productId: 'p1', quantity: 0 }] }, baseContext),
    ).rejects.toThrow('An order needs at least one line');
  });
});
```

---

## 2. Conventions

1. **Inline object-literal fakes** for a port used by one or two tests. Short, readable,
   and you can see exactly what is stubbed.
2. **Extract a fake class** into `tests/fakes/in-memory-*.ts` only when the same fake is
   reused across several test files, **or** when it needs state across calls and is used
   more than once. Reusable fakes drift; inline fakes show intent.
3. **One frozen base context** per file, spread-overridden per test.
4. **Capture side effects in a closure** and assert on the captured value, rather than
   spying on the call. The captured value proves what was written; a spy proves only that
   something was called.
5. **One behaviour per test**, and the test name states it: "trims the title", "rejects a
   cancel on an order that has shipped", "stores a null summary when the generator
   throws".
6. **Cover every branch.** Happy path, every validation failure, not-found, permission
   denial, downstream failure, and each side of every conditional, coalescing operator,
   and early return.
7. **Assert on the typed error and its message**, not just its class. See § 4.
8. **A mocking framework is for services and external integrations.** Repositories get
   in-memory implementations. A mocked repository asserts a method was *called*; an
   in-memory one asserts the *state that resulted*.
9. **No arrange/act/assert comments.** Blank lines separate the phases. The test name and
   the body are the documentation.
10. **No shared mutable state between tests.** Everything is constructed per test.
11. **No network, no database, no file system, no clock.** If the unit under test needs
    one, either it has the wrong dependencies or the test belongs in the integration
    suite.

### 2.1 Builders

Where entities have many fields, a fluent builder per entity beats an inline literal:

```ts
const order = anOrder().forCustomer('cust-1').withLine('p1', 2).placed().build();
```

Builders live in the test tree, are reused before new ones are written, and default
every field to something valid so a test only states what it cares about.

---

## 3. Coverage expectations

- **Every operation of a create/read/update/delete surface has at least one happy-path
  test** that asserts the operation actually changed or returned the right state.
- **Every validation rule and every guarded bad state has a test.** Group identical
  checks behind a shared helper or a parameterised test rather than copying the same
  setup per rule.
- **Use-case tests reach 100% branch coverage.** Assume mutation-testing rigour: every
  conditional, coalescing operator, throw, ternary, and early return has at least one
  test that fails if it is removed or inverted.
- **Generic server-error handling does not need a test.** Catching a downstream failure
  and surfacing a toast is covered by the global error boundary and the error reporter.
  Test the *specific* failures that map to user-visible outcomes.

---

## 4. A green test is not evidence until it can fail

This is the rule that separates a suite from a decoration. A test that stays green when
the behaviour it names is broken proves nothing, and it is worse than no test, because it
reads as coverage.

**Before trusting a green test, make it go red.**

- **Break the thing under test and watch the test fail.** Delete or invert the line the
  test exists for. If it still passes, it is not testing it.
- **Never assert a tautology.** `expect(label).toContain(x)` where `x` is one of the two
  values the code could produce passes whichever branch ran. Assert the value the input
  *forces*, not the set it might be.
- **Never assert against a re-derivation of the code under test.** A result compared to a
  local reimplementation with the same mapping baked in agrees only with itself. Assert
  the value from the specification, or call the real collaborator.
- **A test double must enforce the constraint the real thing enforces**, or the test is
  scoped to what the double omits — and says so.
- **A test that enumerates a set derives the set from a source that grows.** A
  hand-written list of six endpoints does not fail when a seventh is added: it
  under-covers silently while reading as "all of them". Derive from the manifest or
  registry, or assert the length against it.
- **A permission or safety test asserts that the defect it exists to catch is absent**,
  not merely that the happy path works.

---

## 5. Doubles that pass for the wrong reason

The recurring shape, and how to catch it.

### 5.1 The argument-blind lookup

A double whose lookup ignores its argument — returning the one fixture whatever
identifier it is asked for — makes every "which record" assertion vacuous. A lookup that
read the wrong identifier is indistinguishable from one that read the right one.

**Prove each identifier is load-bearing** by mutating the production lookup to ask for a
different one and watching the test go red. If it stays green, the double is blind.

**A double taking more than one identifier must be proven on each separately.** Mutate
one at a time. A double can catch a combined mutation while still ignoring one argument.

**The killing test is usually an allow case, not a denial case.** A blind lookup often
still refuses — throwing the right message for the wrong reason. A suite composed
entirely of denial tests can be entirely vacuous.

### 5.2 Two kinds of "it did not happen"

Do not flatten these; they are proved differently.

- **Never consulted** — validation short-circuits before the lookup runs. Prove with a
  double that **throws if called at all**. A null return cannot distinguish "never ran"
  from "ran and the null was tolerated".
- **Consulted but immaterial** — the lookup runs, but its result does not change what the
  test asserts. The throw probe is *wrong* here; it would fail for an unrelated reason.
  Prove by making the double identifier-aware and showing the mutation still passes.

Label which of the two you have. Merging them loses the distinction that makes the claim
meaningful.

### 5.3 Bare error-class assertions

`rejects.toThrow(ValidationError)` where more than one validation failure can reach the
same call cannot tell the failure it is named for from any other. Assert the message the
input forces, or the specific condition.

---

## 6. Composite keys

To prove each part of a composite key is load-bearing, **pin one part to a constant that
resolves a neighbouring row, and seed that neighbour.**

A mutation that makes the read *miss* proves only that the key is consulted, not that the
part is load-bearing: a miss returns nothing, so every assertion in the block fails at
once and nothing distinguishes the dimensions. A read that lands on a *neighbour* returns
the wrong row, so that dimension's own test fails and the others do not.

Expect the "returns nothing when the key does not match" test to fail alongside the
dimension's own test — it now lands on the pinned neighbour instead of on nothing, so it
is evidence of nothing. **Read the dimension's own named test as the result, never the
count.**

Seed the neighbour. With one row seeded, the mutation degenerates back into a miss and
proves nothing again.

---

## 7. Running and reading the suite

- One command per package, one for the whole repository.
- Targeted runs by test name and by file, so a single class can be run while iterating.
- **A test runner discovers projects by what they reference, not by what they are
  named.** A naming convention needs a new pattern per naming accident, and a project
  that matches none of them silently contributes nothing while reporting "no tests
  found" — the same words an empty project produces.
- **An exact project name selects that project alone.** A search term matching more than
  one is refused, with the names listed. Substring matching silently runs a much larger
  suite than the one that was asked for.
- **Compare the discovered count against the executed count and fail on a difference.**
  A test host can stop part way through and still print a pass summary.
- **Account for every selected test by name.** A count line reads
  `discovered 14 | executed 13 | skipped 1 (ignored: <name>)`. A difference the harness
  cannot name stays a failure.
- **Measure a count baseline on this branch's own merge base**, never a number remembered
  from another branch.
- Output is terse by default: failures with stack traces, successes as counts.

---

## 8. When a number becomes evidence

Any time a measurement is about to be reported — a before/after count, a mutation result,
a timing, a sweep total — check the instrument before the finding.

- [ ] The mutation is confirmed **present in the tree** before any after-count is read.
      An after-count identical to the before-count is the signature of "did not apply",
      not "did not bite".
- [ ] The restore actually reached the file — read the diff, not the exit code. A restore
      can succeed and revert an uncommitted edit of your own on the same file.
- [ ] A two-state comparison separates two *actual* states. Reverting a change that is
      already committed removes nothing, and both sides then measure the same tree.
- [ ] The count's magnitude matches the claimed scope — the right file argument was
      passed.
- [ ] Any process under measurement is the one just built, checked by process id and
      uptime. A health check proves something is listening, not that it is what you
      built.
- [ ] A counter-example that failed, failed the way the hypothesis predicted. **Read the
      failing test names and the error, not the total.** A count that is right by
      accident proves nothing.
- [ ] Every mutation count is reported with its patch width. A shared expression is wider
      than the claim by default.
- [ ] A failure-injection harness uses **non-retryable** error types, and asserts that
      failures observed equals failures injected. A retryable type is retried away and
      the harness reports a plausible smaller number.
- [ ] Commit a checkpoint before starting a measurement round.
