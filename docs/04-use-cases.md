# 04 — Use cases

A use case is one user-facing intent, implemented as one class, exposed as one endpoint.
It is the unit of the system: the unit of authorization, the unit of the contract, the
unit of the test.

---

## 1. The interface

```ts
export interface UseCaseDefinition<TRequest, TResponse> {
  getRequiredRoles(): readonly Role[];
  execute(request: TRequest, context: RequestContext): Promise<TResponse>;
}
```

That is the whole contract. One method for authorization, one for the work.

Reference implementation B's equivalent is a per-use-case interface with a single
`Execute(Request) → Response` method and a response object carrying `successful` plus a
list of typed error values. Same shape, different error convention ([01-principles](01-principles.md) § 7).

---

## 2. The canonical use case

```ts
import { randomUUID } from 'node:crypto';
import { ValidationError, NotFoundError } from '../../common/errors';
import { roles, type Role } from '../../common/roles';
import type { RequestContext } from '../../common/request-context';
import type { OrderEntity } from '../../../domain/entities/order';
import type { OrderRepository } from '../../ports/order-repository';
import type { PricingService } from '../../services/pricing-service';
import type { UseCaseDefinition } from '../use-case-definition';
import { ApiEndpoint } from '../api-endpoint';

export interface CreateOrderRequest {
  customerId: string;
  lines: CreateOrderRequest_Line[];
  note?: string;
}

export interface CreateOrderRequest_Line {
  productId: string;
  quantity: number;
}

export interface CreateOrderResponse {
  order: CreateOrderResponse_Order;
}

export interface CreateOrderResponse_Order {
  orderId: string;
  customerId: string;
  status: 'draft' | 'placed' | 'cancelled';
  totalPence: number;
  lines: CreateOrderResponse_Order_Line[];
  createdAt: string;
}

export interface CreateOrderResponse_Order_Line {
  lineId: string;
  productId: string;
  quantity: number;
  unitPricePence: number;
}

@ApiEndpoint({ namespace: 'orders', method: 'create', roles: ['Editor'], personalData: false })
export class CreateOrderUseCase
  implements UseCaseDefinition<CreateOrderRequest, CreateOrderResponse>
{
  public constructor(
    private readonly orderRepository: OrderRepository,
    private readonly pricingService: PricingService,
  ) {}

  public getRequiredRoles(): readonly Role[] {
    return [roles.editor];
  }

  public async execute(
    request: CreateOrderRequest,
    context: RequestContext,
  ): Promise<CreateOrderResponse> {
    const customerId = request.customerId?.trim();
    if (!customerId) throw new ValidationError('A customer is required.');

    const lines = (request.lines ?? []).filter((line) => line.quantity > 0);
    if (lines.length === 0) {
      throw new ValidationError('An order needs at least one line with a quantity above zero.');
    }

    const priced = await this.pricingService.priceLines(lines);

    const order: OrderEntity = {
      orderId: randomUUID(),
      customerId,
      status: 'draft',
      lines: priced.map((line) => ({
        lineId: randomUUID(),
        productId: line.productId,
        quantity: line.quantity,
      })),
      createdBy: context.userId,
      createdAt: context.nowIso,
    };

    await this.orderRepository.createOrder(order);

    return { order: toContract(order, priced) };
  }
}

function toContract(order: OrderEntity, priced: PricedLine[]): CreateOrderResponse_Order {
  /* ... module-local, not exported ... */
}
```

---

## 3. Rules

### 3.1 Validation lives here

Trim, parse, range-check, and reject in the use case. Repositories assume valid input.

**Order of checks, always:**

1. Shape and value validation → `ValidationError`.
2. Existence of referenced entities → `NotFoundError`.
3. Authorization beyond the role gate (ownership, tenancy) → `ForbiddenError`.
4. Business rules and state machine → `ValidationError` or `ConflictError`.
5. Mutate.
6. Map and return.

Return as early as possible on failure. Do not wrap the happy path in a conditional with
a balancing else — validate, return, and let the happy path run at the top level of the
method.

The example above throws typed errors because that is the shape this reference picked. In
a project that chose the result-object shape
([03-backend-domain-and-ports](03-backend-domain-and-ports.md) § 3.1), each step above
returns a failed response carrying its own error value instead. The order and the
early-return discipline are the same.

**Every failure in that list is a declared outcome.** An unexpected exception — a store
that is unreachable, a bug — is left to propagate and becomes a 500 at the entrypoint.
Do not catch it here to convert it into a declared failure; that hides a defect on the
normal path.

### 3.2 No I/O except through ports

- No vendor SDK client imported into a use case, ever.
- No direct network call.
- No **ambient** clock read. Use `context.nowIso` for the request's single "now", or an
  injected `Clock` port where the use case genuinely needs to read time more than once.
  A global date function is the thing that is banned, not time itself.
- No file-system access.
- Identifier generation via the standard UUID/ULID function is allowed and expected.

### 3.3 Map domain to contract before returning

The entity that came back from the repository is never returned directly. Map it, in a
module-local function in the same file. This is boilerplate, and it is the price of the
API being a black box.

### 3.4 The role declaration appears twice

The decorator's `roles` is metadata read by the **static extractor**. `getRequiredRoles()`
is read by the **runtime dispatcher**. They must agree.

Yes, it is duplication. It is the cost of statically extracting the contract without
executing the code. Add a guard test that fails when they disagree — the extractor has
both values, so the check is cheap.

### 3.5 One responsibility

One use case is one intent. Symptoms that you have two:

- a mode or type discriminator in the request,
- a boolean parameter that switches behaviour,
- a response whose fields are populated in two mutually exclusive groups,
- a name containing "And" or "Or".

Composition happens through services and ports, not by bundling. "Create the order,
email the customer, write the audit entry" is one use case calling three collaborators,
each of which does one thing — not one use case containing three blocks.

### 3.6 Side effects report their own failures

A best-effort side effect (a broadcast, a notification, a cache warm) may leave the
caller successful. It may not leave nobody informed. Either await it and report a
failure, or hand it to a durable queue. See [07-composition-and-entrypoints](07-composition-and-entrypoints.md) § 5.

### 3.7 Collections are paginated

A use case returning a collection that grows with usage exposes page and size (or a
cursor) in its request and response. Decide this when the endpoint is written, not when
a customer's list reaches four hundred rows. See [19-performance-and-scale](19-performance-and-scale.md).

---

## 4. Contract type rules

These are enforced by the extractor and fail the build. Full detail in
[06-api-contract-and-codegen](06-api-contract-and-codegen.md).

1. One named `XxxRequest` and one named `XxxResponse` per endpoint.
2. No map types, no anonymous object literals, and no unresolved generics as a root
   contract type.
3. Nested complex shapes are **named** types. The convention is
   `XxxResponse_Thing`, `XxxResponse_Thing_Part` — the underscore path makes ownership
   obvious in the generated client.
4. No contract type is shared between two endpoints.
5. No domain entity type appears anywhere in the tree.
6. Primitive unions are written out (`'draft' | 'placed' | 'cancelled'`), not aliased to
   a domain type.

### 4.1 Why rule 4 is not negotiable

Two endpoints that share a response type are joined forever. Adding a field for one adds
it to the other, where it is either unpopulated — a lie the type says is true — or
populated by code nobody asked to write. The duplication is visible; the coupling is
not.

---

## 5. Reading data

A read-only use case follows the same shape with two differences: it takes no mutating
dependency, and it maps a list.

```ts
export interface ListOrdersRequest {
  customerId: string;
  cursor?: string;
  limit?: number;
}

export interface ListOrdersResponse {
  orders: ListOrdersResponse_Order[];
  nextCursor: string | null;
}
```

Rules specific to reads:

- A read with no parameters still has a named empty request type. An endpoint with no
  request type is a hole in the generator.
- A read that returns "one or nothing" returns `{ order: X | null }`, not a 404, unless
  the caller genuinely cannot proceed — a page that renders "not found" wants the null,
  a direct fetch of a known id wants the 404. Decide per endpoint and be consistent
  within a namespace.
- Never apply a row limit *before* a filter that runs after the store returns. If the
  store's index is a wider namespace than the filter, the limit applies to the wrong
  set and the caller gets nothing. See [22-pitfalls-checklist](22-pitfalls-checklist.md) § "Collapsing a set".

---

## 6. Recipe: adding an endpoint

1. **Write the contract first.** `ArchiveOrderRequest` and `ArchiveOrderResponse` in the
   use-case file. Nested types named. Nothing reused from another endpoint.
2. **Extend the port** if persistence changes: add `archiveOrder(orderId, by, at)` to
   `OrderRepository`.
3. **Write the use case:** decorator, class, `getRequiredRoles()`, `execute()`.
4. **Implement the port** in the repository. Add the mapper changes.
5. **Wire it** into the namespace `index.ts`.
6. **Write the unit test** beside the use case, with an inline fake of the port. Cover
   every branch: happy path, each validation failure, not-found, permission denial,
   downstream failure.
7. **Regenerate** the contract and clients. Fix any contract violation the extractor
   reports and re-run until it exits zero.
8. **Confirm the generated output contains the new endpoint** — open the file and find
   it. A generator with a hand-maintained template drops what you added with no warning.
9. **Typecheck and build.** A green test suite does not typecheck the code.
10. **Write the end-to-end spec** if the endpoint is reachable from a screen.
11. **Walk the pitfalls checklist** for the areas touched.

The front end can call it as soon as step 7 completes, with full type safety.

---

## 7. Recipe: changing an existing endpoint

Clients lag the server. A mobile build in a user's pocket may be weeks old; a browser tab
may hold a cached bundle. Every contract change keeps old clients working for at least
one release.

| Change | How |
|---|---|
| Add a field | Safe. Optional on the request; always populated on the response. |
| Remove a field | Deprecate: stop reading it, keep populating it for one release, then drop. |
| Rename a field | Add the new one, populate both, drop the old one in a later release. |
| Change a field's type | Never in place. Add a new field; let the old one wither. |
| Rename an endpoint | Keep the old route as a thin forwarder for one release. |
| Tighten roles | Safe. The authorization table is generated at deploy time, so it cannot lag the source. |
| Widen roles | Re-audit the response for data the newly-permitted role must not see. |

**Every one of these needs the generator re-run locally before the typecheck will
see it.** Nothing is committed; the deploy generates its own copy.
