# 02 — Repository layout

One repository holds every deployable surface. The surfaces are separate packages with
their own dependency manifests and their own build, test, and lint commands.

---

## 1. Top level

```
/api                  Backend: domain, application, infrastructure, entrypoints
/web                  Browser single-page application
/mobile               Mobile application
/shared               Contract artifacts shared between packages (generated)
/infra                Infrastructure as code
/scripts              Repository-level operational scripts (migrations, data loads, tooling)
/docs                 Architecture guide, ADRs, runbooks
/.github/workflows    CI pipelines
```

Add only when the project genuinely has one:

```
/site                 Static marketing or content site (no runtime API calls)
/pipeline             Offline data-preparation jobs producing static payloads
/tools                Developer tooling that is not part of a deployable
```

**Rule:** a new top-level directory is an architectural decision. It needs a line in the
architecture guide saying what belongs in it and what does not.

### 1.1 Is this a workspace?

Both reference implementations keep **per-package dependency installs** rather than a
single hoisted workspace. That is a deliberate trade: CI installs more, but a package's
dependency graph cannot be polluted by a sibling, and a package can be built and
deployed alone.

Whichever you choose, **write it down**, because CI depends on it. A pipeline written
for a workspace silently installs nothing for the second package in a non-workspace
repo, and that package's tests then run against a stale build.

---

## 2. The backend package

```
api/
  src/
    domain/
      entities/            Plain data types. No behaviour, no I/O.
    application/
      common/              errors, roles, request-context, error reporting
      ports/               interfaces the application needs from outside
      services/            pure or port-using helpers shared by use cases
      events/              typed event publishers over a generic broadcaster port
      use-cases/           one file per endpoint, plus a per-namespace wiring index
    infrastructure/
      <store>/             base classes, client factory
      repositories/        one implementation per persistence port
      auth/ email/ files/ queue/ search/ ...   one folder per concern
      config/              derived resource names
    entry/
      index.ts             composition root + dispatcher
      routes.<gen>         GENERATED dispatcher table
      auth-context.ts      builds the request context from the transport event
      responses.ts         transport response helpers
      workers/             non-request entrypoints (queue consumers, scheduled jobs)
    local/                 local development host + simulators
    config.ts              environment variables → typed constants, read in ONE place
  tests/
    fakes/                 reusable in-memory port implementations
    integration/           tests against a real-ish store
    setup/                 global test setup (boots the local store once per run)
    guards/                structural tests over IaC, CI, generated artifacts
  scripts/                 contract extraction, client generation, allowlists
```

Notes:

- Unit tests live **next to the file under test** (`create-order-use-case.test.ts`), not
  in a parallel tree. Integration and guard tests live under `tests/` because they are
  about the system, not about one file.
- `config.ts` is the only file that reads environment variables. Everything else takes
  its configuration as a constructor parameter. This is what makes a repository
  instantiable in a test with no environment at all.

### 2.1 Namespaces

Use cases are grouped into **namespaces** — one per bounded area of the API (`orders`,
`documents`, `accounts`). A namespace is a directory under `use-cases/` with its own
`index.ts` that builds every use case in it from injected dependencies:

```ts
export function createOrderUseCases(
  orderRepository: OrderRepository,
  pricingService: PricingService,
) {
  return {
    create: new CreateOrderUseCase(orderRepository, pricingService),
    cancel: new CancelOrderUseCase(orderRepository),
    get:    new GetOrderUseCase(orderRepository),
  };
}

export type OrderUseCases = ReturnType<typeof createOrderUseCases>;
```

A factory function rather than a class, because it is trivially callable from both the
entrypoint (real implementations) and a test (fakes), with no container.

---

## 3. The web package

```
web/
  src/
    api/
      <client>.generated       GENERATED, git-ignored. Never hand-edited.
      core/                    transport: auth token, headers, errors, timeouts
      <feature>Api.ts          hand-written wrappers ONLY for non-JSON transports
    components/                shared and feature components
    pages/                     route-level components
    features/ | <domain>/      feature modules grouped by domain
    hooks/
    layouts/
    config/
    utils/
    test/                      test setup + fixtures
  tests/
    e2e/
      pages/                   page objects, one file per screen
      helpers/                 seeding helpers that call the API
      support/                 auth, reset, mail assertions
      *.spec.ts                specs, one file per feature area
      GUIDE.md                 the e2e rules, copied from this spec set
  <e2e-config>.ts
```

### 3.1 Hand-written API wrappers

The generated client covers every JSON endpoint. Hand-written wrappers exist **only**
for transports the generator does not model: binary upload and download, multipart form
data, streaming. Each such wrapper sits in `src/api/` beside the generated client and is
named for its feature.

**Never hand-declare a type that duplicates a generated one.** It stops tracking the
contract the moment the contract changes, and nothing warns you.

---

## 4. The mobile package

```
mobile/
  src/
    screens/                 one component per screen
    components/              reusable UI
    navigation/
    store/                   state stores, one per concern, plus their tests
    sync/                    sync orchestration, queues, caches, diagnostics
    api/
      <client>.generated     GENERATED, git-ignored. Same generator as the web client.
      client.ts              transport wrapper: auth, retries, base URL
    auth/
    hooks/
    tasks/                   background task registrations
    media/
    config/
    types.ts
  e2e/                       device flow tests
  scripts/                   device harnesses, build helpers
```

See [10-mobile](10-mobile.md) for the rules that govern this package. It has constraints the other two
do not: an offline store that outlives a server deploy, and a platform that suspends the
process for using too much CPU in the background.

---

## 5. Shared contracts

```
shared/
  contracts/
    generated/
      manifest.json          machine-readable endpoint + type manifest
      manifest.ts            the same, as typed source
      manifest.md            human-readable report, including contract violations
```

The manifest is the single source that the client generators read. It is generated from
the backend source. Nothing hand-written lives under `generated/`, and **nothing under
`generated/` is tracked** — the whole directory is ignored.

**Ignore the directory, not the files.** A rule listing each generated file by name stops
covering the next one somebody adds, and that file then lands in a commit. One rule per
generated directory, plus a rule for the `*.generated.*` suffix wherever generated files
sit beside hand-written ones.

**Security note:** where a generated artifact feeds a runtime authorization or privacy
decision, the deploy generates it and **fails** if generation fails. It never falls back
to a previously built copy. See
[06-api-contract-and-codegen](06-api-contract-and-codegen.md) § 6.

---

## 6. Naming conventions

| Kind | File | Symbol |
|---|---|---|
| Domain entity | `domain/entities/order.ts` | `interface Order`, `type OrderEntity = Order` |
| Port | `application/ports/order-repository.ts` | `interface OrderRepository` |
| Use case | `application/use-cases/orders/create-order-use-case.ts` | `class CreateOrderUseCase` |
| Use case test | `.../create-order-use-case.test.ts` | — |
| Namespace wiring | `application/use-cases/orders/index.ts` | `createOrderUseCases` |
| Service | `application/services/pricing-service.ts` | `class PricingService` |
| Event publisher | `application/events/order-event-publisher.ts` | `class OrderEventPublisher` |
| Repository impl | `infrastructure/repositories/<store>-order-repository.ts` | `class <Store>OrderRepository` |
| Reusable fake | `tests/fakes/in-memory-order-repository.ts` | `class InMemoryOrderRepository` |
| Page object | `tests/e2e/pages/order.page.ts` | `class OrderListPage` |
| Screen (mobile) | `screens/OrderDetailScreen.tsx` | `OrderDetailScreen` |

Rules:

- Files kebab-case; classes PascalCase; the file name mirrors the class name.
- The `*Entity` alias on a domain type is a deliberate marker: seeing `*Entity` anywhere
  in a contract is a violation you can grep for.
- Interfaces carry no `I` prefix. The implementation carries the qualifier
  (`InMemoryOrderRepository`, `SqlOrderRepository`), not the interface.
- No abbreviations in identifiers. `quantity`, not `qty`. `address`, not `addr`.

---

## 7. Working-tree hygiene

- The tree is clean before work starts. Uncommitted changes that are not yours are a
  question for the owner, never something to stash or absorb.
- **Generated artifacts never appear in the tree's status at all**, because they are
  ignored. If one shows up as untracked, the ignore rule is wrong — fix the ignore rule
  rather than committing the file.
- A generated file that has been hand-edited is invisible under this rule, because
  nothing tracks it. The protection is that it is **overwritten on the next build**, so a
  hand-edit cannot survive to a deploy. Do not work around that by disabling the
  generation step.

---

## 8. What does NOT go in the repository

- Real secrets, in any form, including in test fixtures and in comments.
- Build output, and every generated artifact named in § 5. All of it is ignored.
- Log files, test result XML, screenshots from ad-hoc runs, editor state. Add them to
  the ignore file the first time one appears; a repository root that accumulates
  hundreds of stray run logs makes every listing useless and hides real files.
- A second project file in a directory that already has one. Any tool that enumerates
  project files finds two and stops.
