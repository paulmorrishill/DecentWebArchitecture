# 07 — Composition and entrypoints

The entrypoint is the only layer that knows about the transport, the environment, and
the vendor SDKs. Everything it does is wiring and translation.

---

## 1. The composition root

One file per deployable. It is the only place that:

- reads environment variables (through the typed `config` module),
- constructs vendor SDK clients,
- constructs concrete infrastructure classes,
- builds the per-request scope,
- builds the use-case bundles,
- dispatches.

```
1. Load config            env → typed constants, validated at startup
2. Construct clients      store, object storage, mail, identity, queue
3. Construct repositories from clients + table names
4. Construct services and event publishers
5. Per request:
   a. build the request scope from verified claims — the ambient-state ports
   b. build the use-case bundles for this request, passing those ports in
   c. resolve the route from the generated table
   d. check roles
   e. execute
   f. map the declared failure or the result to a response
```

Rules:

- **Keep it thin.** Validation, business rules, and store access in the entrypoint is
  what destroys this architecture over time. If logic appears here, push it down.
- **Construct the expensive, stateless things once**, outside the request handler:
  clients, repositories, services. Construct the per-request things per request: the
  ambient-state ports, the tenant-scoped repositories derived from them, and the use-case
  bundles that take them.
- **Use cases are constructed per request.** They are plain objects holding references, so
  this is cheap — and it is what lets each one take exactly the ambient-state ports it
  needs as ordinary constructor parameters. The alternative, a request-scoped proxy
  resolved at call time, is rejected: it puts ambient state back behind an indirection and
  removes the thing the constructor was telling you.
- **Fail at startup on missing configuration.** A configuration value read lazily at
  first use turns a deployment mistake into an intermittent runtime error weeks later.
- **Log which implementation was selected** wherever configuration chooses between a
  real adapter and a no-op. One line in the first log of every deploy.

---

## 2. Building the request scope

The entrypoint constructs one implementation of each ambient-state port
([03-backend-domain-and-ports](03-backend-domain-and-ports.md) § 3.3) and hands them to
the use-case factories. There is no context object, and nothing here is passed to
`execute`.

```ts
export function buildRequestScope(event: TransportEvent, clock: Clock): RequestScope {
  const claims = event.verifiedClaims;               // from the gateway's authorizer
  return {
    caller:   new ClaimsCallerIdentity(claims),
    roles:    new ClaimsCallerRoles(claims),
    tenant:   new ResolvedTenantContext(resolveTenant(event, claims)),
    clock:    new FixedClock(clock.now()),           // one instant for this request
    client:   new HeaderClientInfo(event.headers),
    instance: new HeaderClientInstance(event.headers),
  };
}

// then, per namespace
const orders = createOrderUseCases(orderRepository, pricingService, scope);
```

`RequestScope` is a **construction-time bundle held by the entrypoint**, not a parameter
and not a type the application layer imports. It exists so the factory call has one
argument instead of six. A use case still declares the individual ports it takes; nothing
is allowed to accept the bundle itself.

Rules:

- Claims come from the verified token, never from the body and never from an
  unverified header.
- A header that lets a privileged caller act in another tenant is honoured **only after**
  the role check that permits it. See [08-multi-tenancy](08-multi-tenancy.md).
- **The clock is frozen for the request.** The entrypoint reads the real clock once and
  wraps the value, so every use case in the request agrees. A use case that genuinely
  needs time to advance — a long-running worker loop — takes the real clock instead, and
  says so.
- **Cookies are read and written here, or not at all.** They never reach a use case. See
  [03-backend-domain-and-ports](03-backend-domain-and-ports.md) § 3.3.1.
- **The correlation identifier goes to the error reporter and the logger**, constructed
  here, not to any use case.

---

## 3. Dispatch and authorization

```ts
const action = routes.find((r) => r.namespace === ns && r.method === m);
if (!action) return notFound();

if (!isPublic(action) && !scope.caller.userId()) return unauthorized();
if (!hasAnyRequiredRole(scope.roles.roles(), action.requiredRoles)) return forbidden();

const result = await action.execute(payload);
return ok(result);
```

The dispatcher reads identity and roles from the same ports the use cases use. It does
**not** hand them onward — the use case already holds the ones it declared.

- **Public endpoints are explicit.** The route entry declares the anonymous role; the
  dispatcher computes "is public" from that. An endpoint with an empty role list is a
  bug, not a public endpoint.
- **The role list comes from the generated route table**, which is why that table is
  security-relevant ([06-api-contract-and-codegen](06-api-contract-and-codegen.md) § 6).
- **The privacy gate** runs on the response for endpoints classified as returning
  personal data, before the response leaves.
- **A declared failure is not an error here.** It is a successful call whose response says
  `successful: false` with its error values. The entrypoint returns 200 and does not report
  it. See [03-backend-domain-and-ports](03-backend-domain-and-ports.md) § 3.1.
- **Anything else that reaches the entrypoint becomes a 500**, is reported with its
  request id, and returns a body naming that request id and nothing else. That is the
  designed path for an unexpected exception, not a fallback: a use case is allowed to let
  an unforeseen failure propagate, and the entrypoint is the one place that turns it into
  a response and a report.

---

## 4. Other entrypoints

Every non-request entrypoint gets its own file under `entry/workers/` and its own
composition, reusing the same repositories and use cases.

| Entrypoint | Trigger | Notes |
|---|---|---|
| Queue consumer | A message | Must be idempotent — delivery is at-least-once. |
| Scheduled job | A timer | Must be safe to run twice and safe to skip once. |
| Inbound webhook | A third party | Verify the signature **before** any side effect. |
| Inbound mail | A mail service | Treat the content as untrusted input. |
| Stream or change handler | A store change | Derive scope from the row, not from ambient state. |

Rules for every worker:

1. **A worker builds the same ambient-state ports from different sources**, and then uses
   the same use cases. Read the entity, read its tenant, and construct
   `ConstantTenantContext`, a `SystemCaller` naming the worker, and a real `Clock`.
   Hardcoding a default tenant silently mixes data.

   This is the payoff for having no context object: a worker is no longer a special case
   that "has no context and must fake one". It supplies different implementations of the
   same narrow interfaces, and every use case works unchanged.
2. **Idempotency is designed, not hoped for.** Use a deterministic key derived from the
   entity and the state transition, and a dedupe record where at-least-once delivery
   would break an invariant.
3. **Verify the invariant at the outermost layer that can violate it.** A use case that
   "always writes a terminal status" says nothing about a handler that can throw before
   the use case runs. Read every early return and pre-call path in the wrapper.
4. **A failed message goes to a dead-letter destination and is reported.** A worker that
   silently drops a message is § 6 of the pitfalls list with extra steps.

---

## 5. Background and fire-and-forget work

This is the single most-repeated defect in serverless backends.

### 5.1 A detached promise loses its rejection

A call whose result is discarded — `void doThing()`, an un-awaited promise, a
fire-and-forget task — carries a rejection nobody handles. Worse: explicitly marking it
as intentionally ignored silences the linter's floating-promise rule and takes the
rejection with it.

**Rule:** either `await` it, or attach a handler that reports:
`doThing().catch((err) => reportError('orders.notify.failed', err, { orderId }))`.

### 5.2 On a serverless host, detached work may never run at all

The execution environment is frozen the moment the handler returns its response.
Anything still pending is suspended mid-flight. It does not fail — it stops, and may
resume minutes later on a reused container, or never.

So a handler attached in § 5.1 satisfies that rule while still never publishing, and the
handler never fires either, because nothing rejected.

**Rule:** on a request path, background work is either

- **awaited before the response returns**, or
- **handed to something durable that outlives the invocation** — a queue message, or an
  outbox row written in the same transaction as the state change it describes.

There is no third option. On a long-lived server the detached work usually completes and
the bug is invisible; on a serverless host it is a coin toss decided by container reuse.
That is exactly the failure that reads as "it works locally, and intermittently in
staging".

### 5.3 The outbox

Where the side effect must happen exactly when the state change happens:

1. The use case writes the state change and an outbox row in one transaction.
2. A worker reads outbox rows, performs the side effect, and marks the row done.
3. The worker is idempotent, because it will re-read a row whose side effect succeeded
   and whose mark failed.

Use this when losing the side effect is a correctness failure. Use a plain queue publish
when it is not.

---

## 6. Local entrypoint

The local development host is a **fourth entrypoint with the same composition**: same
use cases, same repositories, different clients pointed at local simulators. Covered in
[14-local-dev-and-simulators](14-local-dev-and-simulators.md).

**It registers its routes from a generated route map**, so an endpoint cannot exist in
the deployed API and be missing locally.

**Keep its differences explicit and few.** Every difference between the local host and
the deployed one is a class of defect the test suite structurally cannot find. Where the
local host fakes a component — token validation, a push channel — write that in the
file's header, and never read a green suite as evidence that the faked component is
correctly wired in the deployed system.

---

## 7. Response helpers

One module, used by every entrypoint.

- Any completed call, successful or not: status 200 and the contract response. A declared
  failure is carried in that body as `successful: false` plus its error values — it is not
  an error at this layer.
- Refused before the use case ran: 401 or 403 from the role gate, 404 for no such route,
  400 for a body that is not the declared shape. A machine-readable code, no prose.
- Unhandled error: 500, the request id, and nothing else. The detail goes to the error
  reporter.
- Cross-origin headers come from one place, derived from configuration, and are covered
  by a guard test against the infrastructure definition — a mismatch between the two is
  invisible until a browser refuses a request in a deployed environment.
