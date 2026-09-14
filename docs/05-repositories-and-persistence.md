# 05 — Repositories and persistence

The infrastructure layer implements the ports. It is the only layer that knows what the
store is.

---

## 1. Shape

One file per port. A small base class holds the shared command helpers; the concrete
repository holds the queries and the mappers.

```ts
// infrastructure/<store>/base-repository.ts
export abstract class BaseRepository {
  protected constructor(protected readonly client: StoreClient) {}

  protected async put(input: PutInput): Promise<void>;
  protected async get(input: GetInput): Promise<GetOutput>;
  protected async query(input: QueryInput): Promise<QueryOutput>;
  protected async queryAll(input: QueryInput): Promise<Row[]>;   // pages internally
  protected async update(input: UpdateInput): Promise<UpdateOutput>;
  protected async deleteOne(input: DeleteInput): Promise<void>;
}
```

```ts
// infrastructure/repositories/store-order-repository.ts
export class StoreOrderRepository extends BaseRepository implements OrderRepository {
  public constructor(
    client: StoreClient,
    private readonly tableName: string,
    private readonly customerIndexName: string,
  ) {
    super(client);
  }

  public async createOrder(record: CreateOrderRecord): Promise<void> {
    await this.put({
      TableName: this.tableName,
      Item: toRow(record),
      ConditionExpression: 'attribute_not_exists(orderId)',
    });
  }

  public async getOrderById(orderId: string): Promise<OrderEntity | null> {
    const result = await this.get({ TableName: this.tableName, Key: { orderId } });
    return result.Item ? toEntity(result.Item as OrderRow) : null;
  }
}

// module-local, never exported
function toRow(record: CreateOrderRecord): OrderRow { /* ... */ }
function toEntity(row: OrderRow): OrderEntity { /* ... */ }
```

---

## 2. Rules

1. **Table and index names are constructor parameters.** A repository never reads an
   environment variable. The composition root resolves configuration in one place. This
   is what lets a test construct the repository with a throwaway table name.
2. **Mappers are module-local.** `toRow` / `toEntity` are not exported. The stored shape
   and the domain shape are decoupled on purpose — that decoupling is how you survive a
   schema change.
3. **Return `null` for not found.** Never throw.
4. **No business logic.** No validation, no defaulting that changes meaning, no
   cross-aggregate reads. If a repository method needs to decide something, the decision
   belongs in the use case.
5. **A fetch-one method is a point read.** Never `listAll().find(...)`. Add a targeted
   method or an index.
6. **A subclass never reaches past the base class to the raw client.** If an operation
   cannot be expressed with the base primitives, extend the base class. A raw client
   call inside a subclass bypasses every guarantee the base class enforces — most
   importantly tenancy ([08-multi-tenancy](08-multi-tenancy.md)).
7. **Bytes go to object storage.** The row holds a key and metadata.
8. **Parameterise every query.** No string concatenation of caller input into a query,
   in any store.

---

## 3. Store-specific hazards

### 3.1 Document / key-value stores

- An attribute that participates in an index and may be absent must be **omitted** from
  the written item, not written as a null. Most stores reject a null on an indexed
  attribute, and the failure arrives at write time in production.
- A query against a secondary index returns rows from the **whole index namespace**. If
  the code filters after the store returns — for tenancy, for status — then any row
  limit passed to the store applies to the *unfiltered* set. Either push the filter into
  the key condition or do not pass a limit.
- Conditional writes are the concurrency primitive. Use a condition expression for
  create-if-absent and for optimistic update, and assert the resulting conflict error in
  a test.
- Batch and transactional writes have different guarantees. A transactional write costs
  more and gives per-item conditions; a batch write is cheaper and gives none. Do not
  "optimise" a transactional write down to a batch write without checking which
  guarantee the code depends on.
- Transactions may be unavailable or differently-shaped in the local simulator. Know
  which, and do not write production code whose correctness depends on a behaviour the
  local store fakes.

### 3.2 Relational stores

- Hand-rolled SQL through a thin mapper is a legitimate choice and is reference
  implementation B's. If you make it, make it consistently: no ORM anywhere, and one
  connection provider that every repository takes.
- Column and table naming is one convention, chosen once. Quote reserved words.
- Cross-table writes happen inside an explicit transaction opened by the calling use
  case, passed down. No ambient transaction scope.
- Connections come from a request-scoped provider. Never construct one inside business
  code.

---

## 4. Primary key shape — decide it explicitly

**Every table's key shape is a decision that goes in the pull request description.** It
is never inherited by accident.

For each new table, state where the key value comes from:

| Origin | Rule |
|---|---|
| Generated UUID/ULID | Safe. Note it and move on. |
| User-chosen name or slug | Must be scoped. See below. |
| User identity | State deliberately whether that identity can repeat across scopes. |
| Externally-owned identifier (a third-party record id, a map feature id) | Shared across scopes by definition. Must be scoped. |

The failure this prevents: two tenants choose the same name, address the same row, and
the second one's write hits an ownership guard that surfaces as a bare 403 with no hint
that a different name would work. The guard is defence in depth. It is not a substitute
for a key that cannot collide.

**Scoping a caller-chosen key:** prefix it with the tenant at the repository boundary
only — encode in `toRow`, decode in `toEntity`. The caller-facing identifier stays plain
everywhere above the repository: entities, use cases, URLs, and the contract must not
change shape.

**A deliberately global namespace** (a public short link, a publicly resolvable slug)
must say so in writing and must return its own error value — `SlugAlreadyTaken` — not an
ownership failure. The client renders one as "choose another name" and the other as "you
are not allowed", and only one of them is true.

**Changing an existing table's key shape strands every existing row.** Read both shapes
during the transition (scoped first, bare second), ship a re-key script alongside, and
delete the fallback once the script reports zero remaining rows in every environment.

---

## 5. Schema is code, and only code changes it

**Never hand-edit schema, create a table by hand, or drop a constraint to get past an
error — in any environment, including local.**

- A missing table means a stale environment. Re-apply the schema.
- A foreign-key or uniqueness violation means the migration, the repository, or the seed
  is wrong. Fix that.
- Destructive schema changes need an explicit instruction from the owner.

Two valid mechanisms:

**A — table definitions in infrastructure-as-code.** Suits a document store where the
"schema" is table names, key attributes, and indexes. Local development reads the same
definitions to create its simulator tables, so the local shape cannot drift from the
deployed one. Put the definitions in one module both sides import.

**B — versioned forward-only migrations.** Suits a relational store. One class per
migration, ordered by an integer, with the real DDL in a sibling `.sql` file loaded as
an embedded resource — SQL reviewed as SQL, not as a fluent chain. `Down` stays empty
unless the owner asks for reversibility. Migrations run from a single command, invoked by
the deploy.

### 5.1 Data changes that are not schema changes

Adding an attribute to a document row is safe. Renaming one is not: existing rows carry
the old name. Either dual-write both names for a release, or run a migration script over
the whole table before the rename ships.

**A backfill script is idempotent and re-runnable.** A single-shot script that crashes
halfway leaves the table mixed with no recovery path. It also reports counts, and — in a
production environment — nothing else. See [21-agent-working-rules](21-agent-working-rules.md) § "Operating on
production".

---

## 6. Test doubles for repositories

Every persistence port gets an in-memory implementation used by unit tests.

```ts
export class InMemoryOrderRepository implements OrderRepository {
  private readonly rows = new Map<string, OrderEntity>();

  async createOrder(record: CreateOrderRecord): Promise<void> {
    if (this.rows.has(record.orderId)) throw new DuplicateKeyError(record.orderId);
    this.rows.set(record.orderId, { ...record });
  }

  async getOrderById(orderId: string): Promise<OrderEntity | null> {
    return this.rows.get(orderId) ?? null;
  }
}
```

Rules:

1. **Implement the port in full**, so adding a method to the port breaks every fake and
   forces a decision.
2. **Enforce the constraints the real store enforces** — uniqueness, conditional writes,
   required attributes. A fake that accepts a write the real store rejects makes the
   test green on code that fails in production. Where the fake genuinely cannot enforce
   something (an index constraint, a transaction), say so and cover it where the real
   store runs.
3. **Answer only for the key it was asked for.** A fake whose lookup ignores its argument
   makes every "which record" assertion vacuous. Prove each identifier is load-bearing
   by mutating the production lookup to ask for a different one and watching the test go
   red.
4. Reference implementation B's rule is worth stating plainly: **a mocking framework is
   for services and external integrations; repositories get in-memory implementations.**
   A mocked repository asserts that a method was *called*; an in-memory one asserts the
   *state that resulted*. The former passes when the guard order is wrong and hides the
   case where a rejected write still mutated something.

---

## 7. Integration tests against the real store

A repository's mappers, condition expressions, and index queries are proven against a
real store — a local emulator or a disposable database. Covered in
[12-testing-integration-and-guards](12-testing-integration-and-guards.md).

What belongs there:

- `toRow` / `toEntity` round-trip.
- Condition expressions producing the right failure.
- Index queries returning the expected rows, in the expected order.
- Pagination across a page boundary.

What does not: business rules. Those are unit tests on the use case.
