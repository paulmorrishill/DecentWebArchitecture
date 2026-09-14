# 06 — API contract and code generation

The contract pipeline is what makes this architecture cohere across the backend, the web
client, and the mobile client. It is the second-most important part of the
specification, after the dependency rule.

---

## 1. The transport

The API is **RPC over HTTP POST**. One path per endpoint, derived from the namespace and
method: `POST /orders/create`, `POST /orders/list`. The body is the request contract as
JSON; the response body is the response contract as JSON.

Why RPC and not REST:

- The unit of the system is the use case, and an RPC path maps one-to-one onto it. A
  REST resource maps onto several, so authorization, contracts, and tests all straddle
  the boundary.
- The contract can be extracted statically from the type system, which is what the whole
  pipeline depends on.
- There is no debate about which verb or which status code expresses a domain outcome.

Declared domain failures ride in the response body as error values on the contract
([03-backend-domain-and-ports](03-backend-domain-and-ports.md) § 3.1), with a 2xx status —
the call succeeded, the answer is "no". Non-2xx is reserved for the role gate, a route that
does not exist, and an unexpected exception. That separation is the third reason to prefer
RPC: the transport status says what happened to the *request*, never what happened in the
*domain*.

**Swap point:** REST or GraphQL can replace this, but then the extractor has to derive
paths and methods from something, and the "one endpoint, one contract pair" rule has to
be preserved by convention instead of by construction. If you swap it, say how those two
things are kept.

---

## 2. The declaration

A metadata-only decorator (or attribute, or annotation) on the use case class:

```ts
export interface ApiEndpointMetadata {
  namespace: string;
  method: string;
  transport?: 'json' | 'binary' | 'multipart';
  roles?: readonly string[];
  personalData: boolean;      // REQUIRED — see § 6.2
}

export function ApiEndpoint(_metadata: ApiEndpointMetadata): ClassDecorator {
  return () => undefined;     // no-op at runtime
}
```

It is a **no-op at runtime**. It exists so a static analysis tool can find endpoints
without executing the code. That matters: executing the code to enumerate endpoints
means constructing every dependency, which means a database, which means the generator
cannot run in CI or on a developer's machine offline.

---

## 3. The pipeline

```
generate:contracts
  ├─ 1. extract   — source → manifest
  └─ 2. generate  — manifest → clients, route table, authorization tables
```

Both steps run as part of the build, so they cannot be skipped.

**Nothing the pipeline emits is committed.** Every output is git-ignored and produced
fresh. That is why generation must be wired into every command that reads an output —
install, typecheck, lint, test, build, dev start, deploy — rather than left to a person
to remember. A fresh clone has no client types until the first generation runs.

### 3.1 Step 1 — extract

A static analysis script (a TypeScript compiler-API wrapper, a Roslyn analyser, a
reflection pass over compiled assemblies — whichever the language makes cheap):

1. Finds every class carrying the endpoint decorator.
2. Resolves the request and response type arguments from the interface it implements.
3. Recursively expands both into a JSON-serialisable manifest of fields, types, and
   nested shapes.
4. Applies the contract rules (§ 4) and records violations.

Outputs, all git-ignored:

- `shared/contracts/generated/manifest.json` — machine-readable
- `shared/contracts/generated/manifest.ts` — the same as typed source
- `shared/contracts/generated/manifest.md` — human-readable report **including the
  violation list**

### 3.2 Step 2 — generate

Reads the manifest and emits:

- `web/src/api/client.generated.ts` — a strongly typed client
- `web/src/api/types.generated.ts` — the contract types
- `mobile/src/api/client.generated.ts` + types — the same, for the mobile transport
- `api/src/entry/routes.generated.ts` — the dispatcher table
- `api/src/local/route-map.generated.ts` — the local development server's route map
- `api/src/entry/<policy>.generated.ts` — any runtime authorization or privacy table
- a generation report listing anything it could not emit

### 3.3 Both steps exit non-zero on a violation

Generation is deliberately **partial-success with visibility**: valid endpoints still
generate, the reports list the problems, and the exit code is non-zero. That keeps a
single bad contract from blocking every other developer while still failing the build.

---

## 4. Contract rules the extractor enforces

1. **The API is a black box.** No domain entity type appears in the contract tree, at
   the root or nested. The `*Entity` naming marker makes this mechanically checkable.
2. **One dedicated named request type and one dedicated named response type per
   endpoint.**
3. **No generic or anonymous root contracts.** No map types, no inline object
   expressions, no unresolved generic parameter as a root type.
4. **Nested complex shapes are named types.** Convention: `XxxResponse_Thing`,
   `XxxResponse_Thing_Part`.
5. **No contract type is shared between endpoints.**
6. **Mapping happens in the use case**, before `execute` returns.
7. **The privacy classification is present and is a literal.** Omitting it is a
   compile error; anything that is not a literal `true`/`false` fails extraction.
8. **The decorator's roles and `getRequiredRoles()` agree.**
9. **Every response declares its own `XxxError` enumeration**, shared with no other
   endpoint, and no contract field carries user-facing prose.

Each rule is a check in the extractor, not a line in a style guide. A rule nobody
enforces is a rule nobody follows.

---

## 5. The generated client

```ts
// what the web code writes
const { order } = await apiClient.orders.create({ customerId, lines });
```

The generated client:

- exposes one method per endpoint, grouped by namespace,
- takes the request type and returns the response type,
- delegates transport to a hand-written core module (auth header, tenancy header, base
  URL, timeouts, error mapping) that the generator does not own,
- is **never hand-edited**, is git-ignored, and is excluded from linting and formatting.

A generator can also emit, per endpoint, a `mock()` factory for component tests to inject
and a `nullResponse()` factory for a cache to initialise with. Both are cheap to generate
and remove a class of hand-written test scaffolding. Adopt them or not per project.

### 5.1 The transport core

Hand-written, one file, owns:

- reading the credential and attaching it to the outbound request,
- attaching the tenancy header where the caller may act across tenants,
- attaching the **client version** and the **client instance identifier**, both on every
  call (see [03-backend-domain-and-ports](03-backend-domain-and-ports.md) § 3.3.2),
- a request timeout — every call has one,
- rehydrating the response into the client's value types before anything else sees it
  (see [25-typed-values-and-serialization](25-typed-values-and-serialization.md) § 6),
- mapping a non-2xx response to the transport failure the client code branches on —
  declared failures arrive with a 2xx and are read from the response body,
- the unauthenticated response: clear the credential, route to sign-in.

The instance identifier is generated **once, here, at client start** — a random value held
in memory. It is not persisted, so a reload or a relaunch produces a new one, which is the
intended behaviour. No screen, hook, or store ever passes it explicitly.

Never put a token in a query string. Headers only.

---

## 6. Generated artifacts are security-relevant

This section is the one people skip and the one that causes incidents.

### 6.1 The route table decides who may call what

The dispatcher looks each request up in the generated route table and reads the required
roles from it. **A stale route table keeps an endpoint open after its roles have been
tightened** — fail-open, and silent, because the source code shows the new roles and the
running system reads the old table.

### 6.2 The privacy table decides what leaves the building

The `personalData` flag on each endpoint is extracted into a generated table the
dispatcher reads at runtime to gate responses that can carry personal data about a real
person. It is required on every endpoint precisely so that a new endpoint cannot default
into the permissive answer.

### 6.3 Therefore: generate at deploy time, and fail closed

Because these files are not committed, there is no stale copy to ship and no diff to
review. What replaces the diff check:

- **The deploy generates them, from the revision being deployed.** They cannot be older
  than the source they came from.
- **Generation failing fails the deploy.** It never falls back to a previously built
  artifact, never reuses a cached one from another revision, and never continues with a
  partially generated table. A missing authorization table means no deploy, not an open
  endpoint.
- **The build artifact contains them**, so what is uploaded and what runs are the same
  files. Do not generate once for the tests and again for the package.
- **A contract violation is a build failure**, at every stage. That is the gate.

### 6.4 Keeping review visibility without committing the output

The one thing a committed artifact gave you was a reviewable diff: a reviewer could see
that tightening a role actually changed the route table. Get that back without tracking
the file.

**Generate a contract-difference report in the pull request check.** The pipeline
generates the manifest for the base revision and for the head revision, compares the two,
and prints the difference as the check's output — endpoints added and removed, roles
changed, privacy classifications changed, contract fields added and removed.

That report is strictly better than the old diff:

- it is written in terms of the contract, not of generated source,
- it cannot be hand-edited into agreement,
- it names the changes a reviewer actually cares about, instead of a few thousand lines of
  regenerated client,
- it is produced by the same extractor the build uses, so it cannot drift from it.

Fail the check on any change to a role or a privacy classification unless the pull request
body names it. That is the review gate, and it does not put a single generated byte in the
repository.

### 6.5 The failure this design removes

Where these artifacts are committed, the subtlest failure in the whole pipeline is an
artifact that did not change. A privacy table that did not change is the correct outcome
for a change that adds no personal data. It is also the exact outcome of a regeneration
that was skipped. **The diff cannot tell the two apart, and only one is safe.** Detecting
it needs a rule about sibling artifacts moving together, and a regeneration to settle the
cases that rule cannot.

Not committing the artifacts deletes that whole problem: there is no "unchanged file" to
interpret, because there is no tracked file at all, and the deployed copy was generated
from the deployed revision.

It is recorded here because it is the reason people commit these files in the first
place — the committed form looks safer and is not. If someone proposes tracking them
again, this is the cost.

---

## 7. Confirm the product, not the exit code

A generator can exit zero and emit less than it was asked to.

- After generating, **open the output and find the symbol you added.** A hand-maintained
  template inside the generator — a per-namespace registration line, an import list —
  does not derive from the manifest and drops what you added with no warning.
- After editing a build or bundle configuration, **list the entries** and confirm the
  ones you did not mean to change are still there. An edit that replaces instead of adds
  drops a bundle silently: the deploy succeeds and the old code keeps running.
- A green test suite does not typecheck the code. **Run the build.**

---

## 8. Versioning and compatibility

There is no API version number. The contract's compatibility rules do the work
([04-use-cases](04-use-cases.md) § 7). Two additional mechanisms:

**Client version.** The client sends its version; the entrypoint exposes it through the
`ClientInfo` port. A use case may branch on it to keep an old client working, and a
guard can refuse a client below a minimum supported version with a dedicated error value
the client renders as an upgrade prompt, rather than a confusing failure.

**Sync payload versioning.** Where a client persists server data locally
([10-mobile](10-mobile.md)), the payload carries a schema version. When the shape changes
incompatibly, bump it; the client discards its local copy and re-fetches rather than
merging two shapes.

---

## 9. Generating other consumers

The same manifest drives anything else that needs the API surface:

- an allowlist for a machine-facing tool server, gating which endpoints it may expose,
- API documentation,
- a contract-difference report between two revisions, for release notes.

Add each as a generator step reading the manifest. Never as a second extractor.

**A tool-server allowlist is an authorization surface.** A call refused by it reports
that the endpoint is not exposed to that client — which is a different failure from a
role denial and must say so, or the next reader spends an hour on the wrong problem.
