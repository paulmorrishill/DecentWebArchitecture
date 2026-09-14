# 03 — Domain and ports

The two innermost layers. Small, boring, and the reason the rest is testable.

---

## 1. Domain entities

Plain data types. One file per aggregate. No methods, no validation, no I/O.

```ts
// domain/entities/order.ts
export type OrderStatus = 'draft' | 'placed' | 'cancelled';

export interface OrderLine {
  lineId: string;
  productId: string;
  quantity: number;
}

export interface Order {
  orderId: string;
  customerId: string;
  status: OrderStatus;
  lines: OrderLine[];
  createdBy: string;
  createdAt: string;
  cancelledBy?: string;
  cancelledAt?: string;
}

export type OrderEntity = Order;
```

Rules:

1. No imports from `application/` or `infrastructure/`.
2. Timestamps are ISO-8601 strings, not date objects. Deterministic in tests, portable
   across stores, and unambiguous on the wire.
3. Identifiers are strings, minted in the use case.
4. The `*Entity` alias is a marker. A grep for `Entity` in the contract manifest finds
   every domain leak.
5. Enumerated values are a closed union of literals. Never a free string.

### 1.1 The alternative: typed identifiers

Reference implementation B goes further and gives every identifier its own type with a
prefix, so an order id cannot be passed where a customer id is expected:

```csharp
public sealed record OrderId : Id
{
    public override string Prefix => "order";
    public OrderId(string value) : base(value) { }
    public OrderId() { }   // generates "order_<ULID>"
}
```

This is a genuine improvement in a language with cheap value types and a serializer that
can be taught the conversion once. It costs a converter, a test-time deterministic-id
hook, and a rule that every new identifier type registers a unique prefix.

**Swap point.** Adopt it or do not, per project, and say which in the architecture guide.
Do not adopt it halfway — a codebase where half the identifiers are typed gives the
reader no signal at all.

### 1.2 Optional values

Pick one representation of absence for the whole backend and enforce it:

- **A** — the language's nullable type, with strict null checking switched on.
- **B** — an explicit `Optional<T>` value type that cannot itself be null and forces a
  `hasValue` check at every call site (reference implementation B's choice, inside the
  application layer; adapters at the boundary may still use nullable types).

Mixing the two produces code where the reader cannot tell which invariant holds.

---

## 2. Ports

A **port** is an interface the application declares for something it needs from outside:
persistence, mail, file storage, a queue, an external API, a clock-adjacent service.

```ts
// application/ports/order-repository.ts
import type { OrderEntity } from '../../domain/entities/order';

export interface CreateOrderRecord {
  orderId: string;
  customerId: string;
  lines: OrderEntity['lines'];
  status: OrderEntity['status'];
  createdBy: string;
  createdAt: string;
}

export interface OrderRepository {
  createOrder(record: CreateOrderRecord): Promise<void>;
  getOrderById(orderId: string): Promise<OrderEntity | null>;
  listOrdersForCustomer(customerId: string): Promise<OrderEntity[]>;
  cancelOrder(orderId: string, cancelledBy: string, cancelledAt: string): Promise<OrderEntity | null>;
}
```

Rules:

1. **Domain verbs, not generic CRUD.** `cancelOrder`, not `update`. A port method named
   for the intent cannot be called with the wrong intent, and its fake is obvious.
2. **Domain entities and local input records only.** No store-specific item shapes, no
   vendor SDK types, no transport types.
3. **`null` for not-found.** A port does not throw for an absent row. Absence is a
   normal answer and the use case decides whether it is an error.
4. **One port per logical aggregate.** Two aggregates in one interface makes every fake
   twice the size it needs to be.
5. **No generic base port.** Resist `Repository<T>`. The specific signature is the
   documentation.
6. **A port is an interface, not a class.** Nothing in `ports/` has an implementation.

### 2.1 Sizing a port

A port with more than about eight methods is usually two ports, or one port plus a read
model. A port with one method and one caller is usually a function.

### 2.2 Ports that are not repositories

Same rules, same folder. Typical set:

| Port | Responsibility |
|---|---|
| `Mailer` | Send a typed message. Not "send this HTML". |
| `FileStore` | Put/get/delete bytes by key; issue a time-limited upload URL. |
| `Broadcaster` | Publish a typed event to connected clients. |
| `QueuePublisher` | Enqueue a typed message for out-of-process work. |
| `UserDirectory` | Resolve identifiers to display names and roles. |
| `IdentityProviderAdmin` | Group membership, password resets, and similar. |

Plus the ambient-state ports of § 3.3, which follow the same rules and live in the same
folder:

| Port | Responsibility |
|---|---|
| `Clock` | Reading time. The only time source. |
| `CallerIdentity` | Who is calling. |
| `CallerRoles` | What they are permitted to be. |
| `TenantContext` | Whose data this request concerns. |
| `ClientInfo` | Which client version is calling, for compatibility branching. |
| `ClientInstance` | An ephemeral identifier for the running client instance. § 3.3.2. |

**The file-bytes rule:** bytes live in object storage; the database row holds a key.
Never a base64 blob in a table row.

---

## 3. Application common

### 3.1 Errors

A closed set, defined once, each carrying its transport status.

```ts
// application/common/errors.ts
export class ValidationError extends AppError  { status = 400; }
export class UnauthorizedError extends AppError { status = 401; }
export class ForbiddenError extends AppError   { status = 403; }
export class NotFoundError extends AppError    { status = 404; }
export class ConflictError extends AppError    { status = 409; }
```

The alternative shape, equally valid, is a result object declaring its failures on the
response type:

```ts
export interface CancelOrderResponse {
  successful: boolean;
  errors: CancelOrderError[];          // 'OrderNotFound' | 'AlreadyShipped' | ...
  order: CancelOrderResponse_Order | null;
}
```

That form puts the failures **in the contract**, so the generated client sees them and the
caller can be made to handle each one. Where the language makes exhaustive matching cheap,
prefer it. Pick one shape per project and never mix them.

Rules, whichever shape you chose:

- Each distinct failure gets its own **value** with its own message. Not one generic
  failure per class. The client needs a real branch to message, and a test needs an
  assertion that only one failure satisfies.
- The message is written for a user, not for a developer. `"An order that has shipped
  cannot be cancelled."` — not `"INVALID_STATE"`.
- A conflict caused by a name or slug someone else already took is a **409 with an
  actionable message**, never a 403. A 403 tells the user they are not allowed; the
  truth is that a different name would work.

### 3.1.1 Unexpected failures are not modelled

An expected failure is one the use case was written to produce. Everything else — an
unreachable store, a null nobody anticipated, a bug — is an **exception, and it is
allowed to propagate**. The entrypoint catches it, reports it with the request id, and
returns a 500 carrying that id and nothing else.

Do not add an `UnknownError` value to the declared set to avoid throwing. It gives the
caller nothing to do and moves a real defect into the normal path, where nobody looks at
it.

The thing to avoid is the middle case: **an ad-hoc error thrown to signal a failure the
use case knew about.** That is an expected outcome dressed as a crash — it reaches the
client as a 500, it cannot be branched on, and it fills the error reports with failures
that are working as designed.

### 3.2 Roles

A closed set of role names, plus two helpers:

```ts
export const roles = { admin: 'Admin', editor: 'Editor', member: 'Member',
                       anonymous: 'Anonymous', superAdmin: 'SuperAdmin' } as const;

export function hasAnyRequiredRole(userRoles: readonly string[],
                                   required: readonly Role[]): boolean;
export function isAdminCaller(userRoles: readonly string[]): boolean;
```

- A wildcard role (`SuperAdmin`) satisfies every requirement. Define it once, here.
- Public endpoints declare the explicit `Anonymous` role rather than an empty required
  list. An empty list is indistinguishable from "somebody forgot".

### 3.3 Ambient request state: injected ports, not a context object

**There is no request-context parameter.** `execute` takes one argument: the request.
Everything a use case knows about *who is calling, when, and on whose behalf* arrives
through constructor-injected interfaces, each one narrow and named for a single concern.

```ts
export interface CallerIdentity { userId(): string; email(): string | null; }
export interface CallerRoles    { roles(): readonly Role[]; has(role: Role): boolean; }
export interface TenantContext  { tenantId(): TenantId; }
export interface Clock          { now(): Instant; }
export interface ClientInfo     { version(): string | null; }
export interface ClientInstance { id(): ClientInstanceId | null; }   // see § 3.3.2
```

A use case takes the ones it uses, and no others:

```ts
export class CancelOrderUseCase implements UseCaseDefinition<CancelOrderRequest, CancelOrderResponse> {
  public constructor(
    private readonly orders: OrderRepository,
    private readonly caller: CallerIdentity,
    private readonly clock: Clock,
  ) {}

  public async execute(request: CancelOrderRequest): Promise<CancelOrderResponse> { … }
}
```

**The constructor is the declaration of what ambient state this use case reads.** That is
the whole point. A single context object lets any use case reach anything without saying
so, and the reader has to read the body to find out; a use case that secretly consults the
caller's roles cannot hide when it has to ask for `CallerRoles` by name.

Rules:

1. **No combined context interface.** If a `RequestContext`-shaped type reappears —
   whatever it is called — everything above is undone and nothing enforces it. This is the
   rule most likely to erode, because recombining them always looks like tidying up.
2. **No resolver, locator, factory, or container** handing these out. That is the same
   defect with an extra step.
3. **Every implementation is built from verified claims**, never from a request body and
   never from a client-supplied header that is not itself verified. See
   [18-security-and-privacy](18-security-and-privacy.md).
4. **A use case taking more than about three of these is doing two things.** Treat it the
   same as any other sizing signal.
5. **`Clock` is the only time source.** There is no request-wide timestamp field to read
   instead. Where every write in one request must carry the same instant, the entrypoint
   injects a `FixedClock` holding the instant it read once — the use case still just asks
   a `Clock`, and the guarantee lives in the implementation rather than in a convention.
6. **The correlation identifier is not a use-case port.** It belongs to the error reporter
   and to logging, injected there at composition. A use case has no reason to know it.
7. **The same ports serve every entrypoint.** A worker injects `SystemCaller`,
   `ConstantTenantContext` and a system `Clock`; an anonymous endpoint injects
   `AnonymousCaller`. A worker is no longer a special case that "has no context" — it
   supplies different implementations of the same interfaces.

### 3.3.1 Cookies, sessions, and client-held state

**None of these reach the application layer.**

- A **cookie** is a transport mechanism. The entrypoint and the gateway own it; the
  application layer never reads or writes one. A machine client or a command-line client
  has no cookies and needs none, and a use case that reads one cannot be called from
  either.
- **Security-bearing identity** comes from the verified token, through `CallerIdentity`
  and `CallerRoles`. Never from a cookie, and never from a request field.
- **Client convenience state** — a previous submission, a draft, a dismissed banner, a
  chosen tab — belongs to the client. The client keeps it in whatever its platform
  provides, and sends it as an **ordinary field on the request** when the server needs it.
  It is untrusted input like every other field, and it is validated like every other
  field.

The consequence worth stating: if some state must not be forged, it does not belong in
client-held storage at all. Put it behind the token, or behind a server-side record the
caller can only reference.

### 3.3.2 The client instance identifier

One piece of client-originated ambient state **is** worth a port, because several features
need it and none of them can get it from the token: an **ephemeral identifier for the
running client instance**.

```ts
export interface ClientInstance { id(): ClientInstanceId | null; }
```

- **The client generates it**, on start, as a random identifier. A browser tab, an app
  launch, a command-line process, a machine-client connection — each is one instance.
- **The client sends it on every call**, as a header, attached once in the transport core
  alongside the credential. No use case and no screen ever passes it explicitly.
- **It is ephemeral by definition.** A new tab, a reload, a relaunch produces a new one.
  There is no promise that it survives anything, and nothing may depend on it doing so.
  `null` is a legitimate answer — a client that does not send one still works.

**What it is for:**

| Use | Why the token cannot serve |
|---|---|
| Suppressing the echo of your own push message | The same user has two tabs open; only one of them made the change |
| Scoping an idempotency key | Two instances of one user may legitimately submit the same thing |
| Addressing a push channel per instance | A notification meant for the tab that started a job |
| Correlating a multi-step flow across calls | The steps belong to one instance, not one user |
| Correlating diagnostics from one session | A support report covers one launch |

**What it is not, and the rules that follow:**

1. **It is not identity, and it is not an authorization input.** It is client-supplied and
   therefore forgeable. Two clients can send the same value. Nothing may be permitted or
   refused because of it.
2. **It is not a device identifier and must not become one.** It dies with the instance by
   design. An identifier that persisted across launches would be a durable handle on a
   person, which is a privacy decision nobody made. Do not persist it client-side, and do
   not store it against a user record.
3. **It carries no meaning.** Not a sequence number, not a hostname, not a user
   identifier with a suffix. A random value, opaque to the server.
4. **It is not a substitute for the correlation identifier**, which the server generates
   per request for tracing. They answer different questions: one identifies the caller's
   process, the other identifies this call.
5. **Treat it as personal-data-adjacent in logs.** It is not a person, but it groups a
   person's actions. It does not belong in a long-retention log or an analytics export.

### 3.4 Error reporting

One function, one signature, available in every package:

```ts
reportError(tag: string, error: unknown, context?: Record<string, unknown>): void;
```

- `tag` is a dotted path naming the site: `orders.cancel.publishFailed`. It is what you
  search for later.
- Every catch block either rethrows or calls this. See [17-observability-and-errors](17-observability-and-errors.md).

---

## 4. Application services

Two kinds live in `application/services/`:

1. **Pure helpers** shared by multiple use cases — date arithmetic, slug generation,
   money rounding, week boundaries. No dependencies. Plain input/output tests.
2. **Thin orchestrators** that use ports but are not themselves endpoints — a statistics
   aggregator, a dispatcher that turns one domain event into several queue messages.

Same dependency rule: a service may use ports, never infrastructure. Every service has a
test file beside it.

**A service is not a dumping ground.** If a "service" ends up with six unrelated methods
and every use case injects it, it is a container in disguise. Split it by the thing it
does.

---

## 5. Events

Typed publishers wrap a generic broadcaster or queue port so publishing is type-safe at
the call site:

```ts
// application/events/order-event-publisher.ts
export class OrderEventPublisher {
  constructor(private readonly broadcaster: Broadcaster) {}

  async publishOrderPlaced(tenantId: string, orderId: string): Promise<void> {
    await this.broadcaster.broadcast(tenantId, { type: 'orderPlaced', orderId });
  }
}
```

Rules:

- The publisher is a **required** constructor parameter of any use case that publishes.
  Never optional, never optional-chained.
- Every broadcast carries the tenant on the envelope. See [08-multi-tenancy](08-multi-tenancy.md).
- The infrastructure layer supplies the concrete broadcaster, including a no-op
  implementation for local development — and composition **logs which one it chose**.
- A publish on a request path is awaited before the response returns, or handed to
  something durable. On a serverless host, detached work is frozen when the handler
  returns. See [07-composition-and-entrypoints](07-composition-and-entrypoints.md) § 5.
