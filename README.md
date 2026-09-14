# Web Application Architecture Specification

A complete, reusable specification for building a web application with an API, a web
front end, a mobile app, infrastructure-as-code, and an automated test estate.

This document set is written **for an AI coding agent** to follow on a new project. A
human can read it too, but the audience that decides the wording is the agent: every
rule is prescriptive, every convention is named, and every trade-off is stated so the
agent does not have to re-derive it.

---

## 1. What this is

This is an **architecture specification**, not a framework and not a template repo. It
describes:

- the shape of the code (layers, boundaries, dependency direction),
- the shape of the contracts between the parts,
- the shape of the tests that prove the code works,
- the shape of the infrastructure and the pipeline that ships it,
- the working rules that keep an agent from producing confident, broken work.

The architecture is **pattern-first**. A concrete stack is named as a *reference
implementation* so the agent has something to write, but every technology choice is
marked as a swap point.

### 1.1 Reference implementations

The pattern below has been implemented twice, in two unrelated stacks. Where the two
differ, that difference marks a swap point rather than a rule.

| Concern | Reference A | Reference B |
|---|---|---|
| Language | TypeScript (Node) | C# (.NET) |
| API host | Serverless function behind an API gateway | Long-lived web host, also deployable as a function |
| Persistence | Document store (key/value + secondary indexes) | Relational database (hand-rolled SQL, no ORM) |
| Schema | Table definitions in IaC | Versioned forward-only migrations |
| Web client | React SPA | Angular SPA |
| Mobile | React Native / Expo | React Native / Expo |
| Unit tests | Vitest | NUnit + FluentAssertions + a single mocking library |
| Web e2e | Playwright | Selenium with page objects |
| Mobile e2e | Maestro flows | Maestro flows |
| IaC | Terraform | Terraform |
| CI | Hosted + self-hosted runners | Hosted + self-hosted runners |

**The parts that are NOT swap points** — change these and the rest stops paying off:

1. The dependency rule (§ [01-principles](docs/01-principles.md)).
2. One use case per endpoint, with its own request and response types (§ [04-use-cases](docs/04-use-cases.md)).
3. The API contract is generated from the code, and the generated artifacts are
   committed and gated (§ [06-api-contract-and-codegen](docs/06-api-contract-and-codegen.md)).
4. Use-case tests use in-memory fakes of ports, not a mocking framework aimed at
   repositories (§ [11-testing-unit](docs/11-testing-unit.md)).
5. End-to-end tests drive the real UI against a real backend with no faked API
   responses (§ [13-testing-e2e](docs/13-testing-e2e.md)).

---

## 2. How to use this document set

### 2.1 Starting a new project

Read in this order, and do the work in this order:

1. [01-principles](docs/01-principles.md) — the rules that everything else follows from.
2. [02-repository-layout](docs/02-repository-layout.md) — create the directories and the package boundaries first.
3. [15-infrastructure-and-environments](docs/15-infrastructure-and-environments.md) — decide environments and naming before code.
4. [03-backend-domain-and-ports](docs/03-backend-domain-and-ports.md) → [04-use-cases](docs/04-use-cases.md) → [05-repositories-and-persistence](docs/05-repositories-and-persistence.md)
   — build one vertical slice end to end.
5. [06-api-contract-and-codegen](docs/06-api-contract-and-codegen.md) — stand the generator up as soon as the second
   endpoint exists, not later.
6. [14-local-dev-and-simulators](docs/14-local-dev-and-simulators.md) — make the whole stack runnable offline before adding
   the third feature.
7. [09-frontend](docs/09-frontend.md), then [10-mobile](docs/10-mobile.md).
8. `11`–`13` testing — write the first test of each kind alongside the first feature,
   never as a later pass.
9. [16-ci-cd](docs/16-ci-cd.md).
10. [20-decision-records](docs/20-decision-records.md), [21-agent-working-rules](docs/21-agent-working-rules.md), [22-pitfalls-checklist](docs/22-pitfalls-checklist.md) —
    copy these into the new repo as living documents on day one.

### 2.2 Adding a feature to an existing project

[23-new-feature-recipe](docs/23-new-feature-recipe.md) is the checklist. It links to the other files.

### 2.3 What to copy into the new repo

Copy these into the new repository and keep them updated as the project learns:

| File in this set | Lands in the repo as |
|---|---|
| [21-agent-working-rules](docs/21-agent-working-rules.md) | `AGENT_RULES.md`, or the agent instructions file the harness reads |
| [22-pitfalls-checklist](docs/22-pitfalls-checklist.md) | `PITFALLS.md` — append a section every time a class of defect reaches a deployed environment |
| [20-decision-records](docs/20-decision-records.md) | `docs/adr/README.md` + `docs/adr/_template.md` |
| `04`, `05`, `06` | `docs/architecture-guide.md` |
| [13-testing-e2e](docs/13-testing-e2e.md) | `<web>/tests/e2e/GUIDE.md` |

The rest stay as the specification they are.

---

## 3. The index

| File | Covers |
|---|---|
| [01-principles.md](docs/01-principles.md) | Dependency rule, black-box contracts, sizing, what "done" means |
| [02-repository-layout.md](docs/02-repository-layout.md) | Monorepo shape, package boundaries, naming, generated files |
| [03-backend-domain-and-ports.md](docs/03-backend-domain-and-ports.md) | Entities, ports, typed errors, request context, clock |
| [04-use-cases.md](docs/04-use-cases.md) | The use case pattern, contracts, validation, roles, recipe |
| [05-repositories-and-persistence.md](docs/05-repositories-and-persistence.md) | Repository pattern, mappers, key shape, schema changes |
| [06-api-contract-and-codegen.md](docs/06-api-contract-and-codegen.md) | Decorator metadata, extraction, generation, contract rules, compatibility |
| [07-composition-and-entrypoints.md](docs/07-composition-and-entrypoints.md) | Composition root, dispatcher, workers, background work |
| [08-multi-tenancy.md](docs/08-multi-tenancy.md) | Tenant scoping, key shapes, resolvers, isolation tests |
| [09-frontend.md](docs/09-frontend.md) | SPA structure, generated client, state, forms, errors, design system |
| [10-mobile.md](docs/10-mobile.md) | App structure, store, offline sync, media, background cost |
| [11-testing-unit.md](docs/11-testing-unit.md) | Unit doctrine, fakes, builders, branch coverage, mutation discipline |
| [12-testing-integration-and-guards.md](docs/12-testing-integration-and-guards.md) | Real-store tests, structural guard tests over IaC and CI |
| [13-testing-e2e.md](docs/13-testing-e2e.md) | Page objects, test ids, seeding, simulators, the no-mock rule |
| [14-local-dev-and-simulators.md](docs/14-local-dev-and-simulators.md) | Running the whole stack with no cloud account |
| [15-infrastructure-and-environments.md](docs/15-infrastructure-and-environments.md) | IaC layout, environments, naming, destructive-change review |
| [16-ci-cd.md](docs/16-ci-cd.md) | Pipelines, gates, artifact checks, deploys, releases |
| [17-observability-and-errors.md](docs/17-observability-and-errors.md) | Error taxonomy, reporting, no silent swallow, startup logging |
| [18-security-and-privacy.md](docs/18-security-and-privacy.md) | Authorization model, personal-data gate, secrets, least privilege |
| [19-performance-and-scale.md](docs/19-performance-and-scale.md) | Pagination, N+1, judging measured latency, retry and backoff |
| [20-decision-records.md](docs/20-decision-records.md) | ADR process and template |
| [21-agent-working-rules.md](docs/21-agent-working-rules.md) | How an agent works in the repo: branches, commits, reporting, verification |
| [22-pitfalls-checklist.md](docs/22-pitfalls-checklist.md) | The pre-completion checklist of recurring defect shapes |
| [23-new-feature-recipe.md](docs/23-new-feature-recipe.md) | End-to-end recipe from spec to merged |
| [24-glossary.md](docs/24-glossary.md) | Terms used across the set |

---

## 4. The five sentences that matter most

If the agent reads nothing else:

1. **The application layer never imports the infrastructure layer.** That one rule is
   what makes the use cases testable in milliseconds and the infrastructure swappable.
2. **Every endpoint owns its request and response types, and shares them with nobody.**
   It looks like duplication. It is the reason a change to one endpoint cannot silently
   change another.
3. **The API contract, the client, and the route table are generated from the code and
   committed.** A hand-edited generated file is a security incident waiting to be read
   as a merge conflict.
4. **A test is not evidence until it has been seen to fail for the right reason.** A
   green suite over a vacuous assertion is worse than no suite, because it reads as
   coverage.
5. **Report what is true.** A non-2xx response, a red check, a failed test, a thrown
   error is a failure — fix it, or report it plainly at the top of the message. Never
   under a green tick.
