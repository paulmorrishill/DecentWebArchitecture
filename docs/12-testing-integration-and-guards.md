# 12 — Integration tests and structural guards

Two suites that answer questions a unit test cannot: does the persistence layer really
work, and does the code agree with the things outside it.

---

## 1. Integration tests

**Subject:** the repository against a real store. Not business logic.

### 1.1 Setup

A global setup boots the store once per run, before any test file loads, and exports its
endpoint through the environment:

```ts
export default async function setup(): Promise<() => Promise<void>> {
  const port = await findFreePort();
  server = startLocalStore({ port });
  process.env.STORE_ENDPOINT = `http://localhost:${port}`;
  return async () => { await server.close(); };
}
```

Two valid shapes:

- **An in-process emulator** for a document store. Fast, no container, boots in
  milliseconds. Register the setup in **both** test configurations, because some
  repository tests are cheap enough to live beside the repository as unit tests.
- **A real database in a container** for a relational store. Each test gets a freshly
  named database, created and migrated. Reset the connection pool between tests, or
  stale handles point at dropped databases.

### 1.2 Container discipline, where containers are used

- **One named, version-pinned container.** Never a floating latest tag, and the same pin
  as the local development configuration and CI.
- The harness **replaces** the container by default, so a run starts from an empty
  server. Reuse is opt-in and the harness **says** it reused. Silent reuse is not
  allowed: a run inheriting the last run's schema must announce it.
- The harness refuses to start when a container it did not create holds the port. It
  names that container and stops. It never removes a container it did not create.
- The harness reports free memory and orphaned build workers at the start and end of a
  run, and reaps what it can. Build tooling that reuses worker processes leaves them
  alive after the build that started them; left alone they accumulate until memory
  exhaustion produces failures that read as failing hardware.

### 1.3 What belongs here

- Mapper round-trip: entity → row → entity.
- Conditional writes producing the right conflict.
- Index queries returning the expected rows in the expected order.
- Pagination across a page boundary.
- Behaviour the in-memory fake cannot enforce — index constraints, transaction
  semantics, the store's own validation.

### 1.4 What does not

Business rules. Those are unit tests on the use case. An integration suite that
re-proves validation is slow, duplicated, and the first thing that rots.

### 1.5 Tests own their own data

Each test creates uniquely named tables or a uniquely named database, and does not depend
on another test's rows. Skip the suite cleanly when the store endpoint is absent, so the
unit suite still runs on a machine that cannot start it.

---

## 2. Structural guard tests

**Subject:** agreement between things that live in different languages and cannot
reference each other — application code, infrastructure definitions, CI configuration,
generated artifacts.

These are ordinary unit tests in the backend package that read files from elsewhere in
the repository. They are cheap, they run in the normal suite, and they catch a class of
defect that nothing else can: a setting that drifted, where both sides are individually
valid.

### 2.1 The guards worth having

| Guard | Catches |
|---|---|
| Every scoped table declared in code has its tenant attribute and tenant index in the infrastructure definitions | A runtime index-missing failure that only appears in a deployed environment |
| Every scoped repository has an isolation fixture | A new table added with no cross-tenant test |
| Identity-keyed tables match the documented allowlist | A key shape that silently collides across tenants |
| Cross-origin origins in the infrastructure match the ones the code emits | A browser refusing requests after deploy |
| The endpoints a machine-facing tool server exposes match the generated allowlist | An endpoint exposed to a client that should not reach it |
| Privacy classification present on every endpoint, and the generated table matches | An endpoint defaulting into the permissive answer |
| Pipeline concurrency and path filters are what the repository intends | A workflow that silently stops running, or cancels the run that mattered |
| Deployment path filters cover every deployable | A package that stops deploying because its path was not listed |
| Dependency direction: no import from the application layer to the infrastructure layer | The one rule everything else depends on |

### 2.2 Parse the subject; never quote a line

**A guard that parses its subject survives a refactor. One that quotes a literal line is
a tripwire.**

Walk the configuration structure, or anchor on a stable key. A guard that demands a
literal value will fail the day someone replaces that literal with the derived expression
the guard existed to encourage — the resolved value identical, the behaviour intact, the
build red.

That is the trap worth naming: a guard broken by reformatting is a nuisance; **a guard
that fails the correctness fix it was written to encourage will be read as evidence the
fix is wrong.**

### 2.3 A guard changes which tests gate which files

Once a guard in the backend package reads infrastructure or pipeline files, a change to
**only** those files can break the backend suite. So:

- A pull request touching infrastructure or CI configuration runs the backend unit tests
  before merge, even when it changes no application code.
- Say so in the repository's contribution rules, because it is not obvious.

---

## 3. Contract-artifact checks in CI

Distinct from both suites and run on **every pull request**, because the deploy may not
wait for anything else:

1. Run the generator.
2. Fail on any diff in the generated artifacts.

This catches the change that edited a decorator and did not regenerate. Where a generated
artifact feeds a runtime authorization or privacy decision, this check is the only thing
between a source change and a deployed system that still enforces the old rule.

See [06-api-contract-and-codegen](06-api-contract-and-codegen.md) § 6.

---

## 4. Verify the product, not the exit code

A suite can exit zero having run less than you think.

- **Check the file count and the test count against an expected total**, not the exit
  code alone. A suite reporting "164 passed" while three files failed to load is green
  and missing a third of its coverage.
- **For a file you added, find its own line in the output** confirming it ran. That a file
  loaded is a separate fact from that the total is green.
- **A passing suite does not typecheck the code.** Most test runners strip types rather
  than checking them, so a suite goes green over code the compiler would reject. Run the
  build and confirm the artifact.
- **Read first-attempt failures separately from the final counts.** A run that exits zero
  with three tests failing their first attempt and passing on retry has hidden a real
  defect. A flaky test is a defect to triage, not a pass.

---

## 5. Test environment parity

Every difference between the test environment and a deployed one is a class of defect the
suite structurally cannot find. Keep the list short, and keep it written down:

- Which components are simulated locally, and what that means they cannot prove.
- Which environment variables CI supplies to each package's build. A front-end feature
  gated on configuration CI does not supply never mounts in CI, so its component tests
  must mock the configuration module or they prove nothing.
- Which store features the local emulator lacks, and where the real-store tests cover
  them instead.

**Never read a green suite as evidence about a component the suite replaces with a
double.** A double cannot fail to be wired, so a suite built on one cannot catch a real
dependency that was not.
