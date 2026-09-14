# 04 — Use cases

A use case is one user-facing intent, implemented as one class, exposed as one endpoint.
It is the unit of the system: the unit of authorization, the unit of the contract, the
unit of the test.

---

## 1. The interface

```ts
export interface UseCaseDefinition<TRequest, TResponse> {
  getRequiredRoles(): readonly Role[];
  execute(request: TRequest): Promise<TResponse>;
}
```

That is the whole contract. One method for authorization, one for the work.

**`execute` takes one argument.** There is no request-context parameter. Who is calling,
when, and on whose behalf all arrive through constructor-injected ports
([03-backend-domain-and-ports](03-backend-domain-and-ports.md) § 3.3), so a use case's
constructor states exactly which ambient state it reads.

A synchronous `Execute(Request) → Response` is the same shape in a language where the I/O
boundary is synchronous. Nothing else about the interface changes.

---

## 2. The canonical use case

```ts
import { randomUUID } from 'node:crypto';
import { roles, type Role } from '../../common/roles';
import type { OrderEntity } from '../../../domain/entities/order';
import type { OrderRepository } from '../../ports/order-repository';
import type { PricingService } from '../../services/pricing-service';
import type { CallerIdentity, Clock } from '../../ports/request-scope';
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

export type CreateOrderError =
  | 'CustomerRequired'
  | 'CustomerNotFound'
  | 'NoLinesWithQuantity'
  | 'ProductUnavailable';

export interface CreateOrderResponse {
  successful: boolean;
  errors: CreateOrderError[];
  order: CreateOrderResponse_Order | null;
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
    private readonly caller: CallerIdentity,
    private readonly clock: Clock,
  ) {}

  public getRequiredRoles(): readonly Role[] {
    return [roles.editor];
  }

  public async execute(request: CreateOrderRequest): Promise<CreateOrderResponse> {
    const customerId = request.customerId?.trim();
    if (!customerId) return failed(['CustomerRequired']);

    const lines = (request.lines ?? []).filter((line) => line.quantity > 0);
    if (lines.length === 0) return failed(['NoLinesWithQuantity']);

    const priced = await this.pricingService.priceLines(lines);
    if (priced.unavailable.length > 0) return failed(['ProductUnavailable']);

    const order: OrderEntity = {
      orderId: randomUUID(),
      customerId,
      status: 'draft',
      lines: priced.map((line) => ({
        lineId: randomUUID(),
        productId: line.productId,
        quantity: line.quantity,
      })),
      createdBy: this.caller.userId(),
      createdAt: this.clock.now(),
    };

    await this.orderRepository.createOrder(order);

    return { successful: true, errors: [], order: toContract(order, priced) };
  }
}

function failed(errors: CreateOrderError[]): CreateOrderResponse {
  return { successful: false, errors, order: null };
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

1. Shape and value validation → its own error value (`CustomerRequired`).
2. Existence of referenced entities → its own error value (`CustomerNotFound`).
3. Ownership and tenancy, beyond the role gate → its own error value (`NotYourOrder`).
4. Business rules and the state machine → its own error value (`OrderAlreadyShipped`).
5. Mutate.
6. Map and return a successful response.

Return as early as possible on failure. Do not wrap the happy path in a conditional with
a balancing else — validate, return, and let the happy path run at the top level of the
method.

**Every one of those is a declared value in the response's error enumeration, and none of
them carries a message.** The backend does not know the reader's language
([03-backend-domain-and-ports](03-backend-domain-and-ports.md) § 3.1.1). Where the client
needs data to build its message — the conflicting name, the allowed range — that goes in a
named field on the response, not in interpolated prose.

**An unexpected exception is left to propagate** and becomes a 500 at the entrypoint. Do
not catch it here to convert it into a declared failure; that hides a defect on the normal
path.

The validation this rule is about is **business validation**. A request that is not even
the declared shape — a missing required field, a string where a number belongs — is
rejected by the transport before the use case runs, and needs no error value.

### 3.2 No I/O except through ports

- No vendor SDK client imported into a use case, ever.
- No direct network call.
- No **ambient** clock read. Time comes from the injected `Clock`. A global date function
  is the thing that is banned, not time itself.
- No file-system access.
- No cookie, header, or transport detail of any kind. Those belong to the entrypoint.
- Identifier generation via the standard UUID/ULID function is allowed and expected.
  Inject a generator instead where a test needs determinism.

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

### 3.5 A use case serves one actor

**An actor is whoever can change their mind on their own** and ask for this behaviour to
be different. A person in a role, or another system. Not a screen, not a table, not a
framework, and not a class.

**One use case serves exactly one actor.** Two actors means two independent reasons for
the same code to change, and a change requested by one of them then has to be made without
breaking the other — in a file neither of them owns.

Actors are roles, not people. One person can be two actors at different moments, with
different interests, and that still splits the use case.

#### The test

For the use case in front of you, **name the one actor who can request a change to it.**

| Names | Verdict |
|---|---|
| One | Correct. |
| Two or more | Split it, or write down why you are not and what it costs. |
| None | It is not a use case. It is a service, a mapper, or a helper. |

#### Deriving use cases from actors

When a feature arrives as prose, do this before writing any contract:

1. List the actors.
2. For each actor, list the work they ask the system to do.
3. Write each item as `<Actor> <verb> <object>` — "Reviewer approves submission",
   "Scheduler cancels expired hold".
4. One use case per item.
5. Check each one against the test above, and split where two names appear.

This produces the endpoint list, the names, and the authorization boundaries in one pass,
and it produces them from the domain rather than from the screen that happens to be
getting built first.

#### The failure it prevents

The shape is always the same: one class named for a *thing* rather than for a *request* —
`ManageOrder`, `HandleBooking`, `ProcessSubmission`. Several actors' needs accumulate in
it, each guarded by a flag, and eventually a change for one actor breaks another's path
because the two share a branch neither of them asked for.

#### Secondary signals

Once you have the actor test, these are symptoms of failing it rather than rules of their
own:

- a mode or type discriminator in the request,
- a boolean parameter that switches behaviour,
- a response whose fields are populated in two mutually exclusive groups,
- a name containing "And" or "Or",
- a name containing "Manage", "Handle", or "Process",
- a constructor taking more than about three of the ambient-state ports.

#### Splitting is not bundling

Composition happens through services and ports, not by bundling. "Create the order, email
the customer, write the audit entry" is **one** use case — one actor, the customer, asking
for one thing — calling three collaborators that each do one thing. It is not three
blocks in one method, and it is not three use cases.

The question is never how many steps the work takes. It is how many people can ask for the
work to be different.

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
7. **Every response declares its own error enumeration**, named `XxxError`, owned by that
   endpoint and shared with nothing. A shared error enumeration couples two endpoints'
   failure modes exactly as a shared response type couples their data.
8. **No field in any contract carries user-facing prose.** The extractor cannot check this
   mechanically, so it is a review question on every new contract.

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
- **A read that finds nothing is not a transport failure.** It returns a successful
  response with a null payload, or with an error value where the caller needs to
  distinguish "does not exist" from "exists and you may not see it". Never a 404 — the
  transport's 404 means the endpoint does not exist, and a client cannot tell the two
  apart. Decide per endpoint which of the two shapes you want, and be consistent within a
  namespace.
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
