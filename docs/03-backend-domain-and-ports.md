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
| `Clock` | Only if something needs time beyond the request's "now". |
| `IdentityProviderAdmin` | Group membership, password resets, and similar. |

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

Rules:

- Each distinct failure gets its own error **value** with its own message. Not one
  generic error per class. The client needs a real branch to message, and a test needs
  an assertion that only one failure satisfies.
- The message is written for a user, not for a developer. `"An order that has shipped
  cannot be cancelled."` — not `"INVALID_STATE"`.
- A conflict caused by a name or slug someone else already took is a **409 with an
  actionable message**, never a 403. A 403 tells the user they are not allowed; the
  truth is that a different name would work.

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

### 3.3 Request context

Every use case receives this as its second argument. It is the only channel through which
ambient request state reaches business logic.

```ts
export interface RequestContext {
  requestId: string;
  userId: string;
  userEmail: string | null;
  userRoles: readonly string[];
  tenantId: string;              // see 08-multi-tenancy
  nowIso: string;                // the single source of "now"
  clientVersion?: string;        // for compatibility decisions
  cookies?: Record<string, string>;
  setCookies?: string[];         // mutable; use cases append
}
```

Rules:

- `nowIso` is set once, at the entrypoint, per request. A use case never calls the
  clock.
- The context is **built from verified claims**, never from a request body or a
  client-supplied header that is not itself verified. See [18-security-and-privacy](18-security-and-privacy.md).
- Do not put a service, a repository, or a logger on the context. Those are constructor
  parameters. The context is data.

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
