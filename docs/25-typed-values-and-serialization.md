# 25 — Typed values, the wire format, and rehydration

JSON has no date, no time, no duration, no decimal, no optional, and no identity. Every
one of those has to be carried as a string or a number and turned back into a real value
at the other end.

**Where that conversion is not specified once and generated, it gets re-derived by hand at
every call site, and the derivations drift.** This file specifies it.

---

## 1. The two strategies

Pick one **per project**, write it in an ADR, and hold to it. Mixing them produces a
codebase where the reader cannot tell what a field means without opening the mapper.

### Strategy A — primitives everywhere

Domain types carry ISO-8601 strings, integer minor units, and plain strings. Nothing is
converted anywhere: the store holds the string, the contract holds the string, the client
holds the string, and any arithmetic happens in a named helper.

- **Buys:** no converter layer, no library, nothing to get wrong on the round trip,
  trivially deterministic tests.
- **Costs:** the type system stops helping. A wall-clock time and an instant are the same
  type. Nothing stops you subtracting two unrelated dates. Formatting and comparison logic
  spreads.
- **Suits:** a small surface, a document store, a team that would rather keep the
  boundary thin than model time precisely.

### Strategy B — rich value types with generated conversion

The domain uses a proper date-and-time library and other value types — instant, local
date, local time, zoned date-time, duration, money, typed identifier, optional. The wire
format is specified per type. Converters do the translation at each boundary, and **the
generator emits the client-side rehydration** so no consumer parses by hand.

- **Buys:** a wall-clock time cannot be used as an instant; a date-only value cannot be
  shifted by a timezone; money cannot be a float; an identifier of one kind cannot be
  passed where another is expected. The compiler enforces the distinctions the domain
  actually has.
- **Costs:** a converter registry, a generator that knows about semantic types, a
  round-trip test per type, and a rule for what happens when a converter changes.
- **Suits:** anything with scheduling, availability, recurrence, billing, or
  multi-timezone users. If the product has a calendar in it, this is the answer.

The rest of this file is Strategy B. Under Strategy A, read § 2 and § 8 and skip the
rest.

---

## 2. The wire format is a specification, not an implementation detail

One table, in the architecture guide, filled in before the second endpoint exists. It is
the contract for every client, present and future.

| Semantic type | Wire form | Example | Notes |
|---|---|---|---|
| Instant (a point in time) | string, ISO-8601, UTC, `Z` suffix | `"2026-03-01T09:30:00Z"` | Always UTC on the wire. Never a local offset. |
| Local date (no time, no zone) | string, `YYYY-MM-DD` | `"2026-03-01"` | **Never** an instant. See § 7. |
| Local time (wall clock) | string, `HH:mm` or `HH:mm:ss` | `"09:30"` | No zone, no date. |
| Zoned date-time | object: local date-time + zone id | `{ "local": "2026-03-01T09:30:00", "zone": "Europe/London" }` | Only where the zone is part of the meaning. |
| Duration / span | integer seconds (or milliseconds — pick one) | `5400` | Never a formatted string. Never "1h 30m". |
| Date range | object of two local dates, both inclusive or both half-open — say which | `{ "from": "...", "to": "..." }` | The convention goes in this table, not in each endpoint. |
| Money | integer **minor units** plus a currency code | `{ "amount": 1995, "currency": "GBP" }` | Never a float. Never a formatted string. |
| Decimal (non-money) | string | `"0.1"` | A JSON number is a float. If the precision matters, it is a string. |
| Large integer | string | `"9007199254740993"` | Beyond 2^53 a JSON number loses precision in a browser. |
| Typed identifier | string, prefixed | `"order_01HQ..."` | The prefix is the type marker. See § 4. |
| Optional value | **field omitted** when absent | — | Not `null`. See § 5. |
| Enumeration | string, the literal union member | `"placed"` | Never an ordinal integer. |
| Binary | never on the wire | — | Object storage, with a time-limited URL. |

Rules for the table:

- **One representation per semantic type, across the whole API.** Two endpoints
  disagreeing about how a duration is carried is a defect, not a style difference.
- **Changing a row is a breaking change** for every client and every persisted copy. It
  goes through the compatibility procedure in `04-use-cases` § 7, with a new field rather
  than a changed one.
- **The table is the thing the generator reads.** It is not documentation of what the code
  happens to do.

---

## 3. The conversion happens at three boundaries, and only there

```
  store  ──[store converter]──► domain ──[wire converter]──► JSON
                                  ▲                            │
                                  │                     [generated client]
                                  │                            ▼
                               domain  ◄──[rehydration]──  client value
```

| Boundary | Owner | Direction |
|---|---|---|
| Store ↔ domain | the repository's `toRow` / `toEntity` mappers | both |
| Domain ↔ wire | the serializer's converter registry | both |
| Wire ↔ client value | the **generated** client's rehydrate / dehydrate functions | both |

Nothing else converts. In particular:

- **A use case never parses a string into a value type.** It receives values already
  converted.
- **A component never parses a wire string.** It receives values already rehydrated.
- **A repository never writes a wire-format value** because it happens to match. The store
  format and the wire format are two independent decisions (§ 8).

---

## 4. The converter registry

One module. One converter per semantic type, registered once with the serializer, applied
by type rather than by field name.

```ts
registerConverter(Instant,   toIso,        fromIso);
registerConverter(LocalDate, toYmd,        fromYmd);
registerConverter(Duration,  toSeconds,    fromSeconds);
registerConverter(Money,     toMinorUnits, fromMinorUnits);
```

Rules:

1. **By type, never by field name.** A converter keyed on `createdAt` misses
   `cancelledAt`, and misses every new field somebody adds.
2. **Identifiers convert generically.** One converter that handles every typed identifier
   by reading its prefix, so adding a new identifier type needs no new converter. This is
   the single biggest reason typed identifiers stay cheap.
3. **Bidirectional and total.** Every converter round-trips: `from(to(x)) == x` for every
   value in the type's domain, including the edges — a leap day, a daylight-saving
   transition, the zero duration, the maximum representable instant.
4. **Converters never throw for an expected input.** An unparseable value arriving from a
   client is a declared failure on the endpoint — its own error value — not a 500.
5. **One serializer for the whole backend.** Two serializers with two registries is the
   same defect as two wire formats.

---

## 5. Optional, null, and absent

Three states the wire can express, and the project uses exactly two:

| Meaning | Wire |
|---|---|
| The value is absent | the field is **omitted** |
| The value is present | the field carries it |
| ~~The value is explicitly null~~ | **not used** |

Reserve `null` for one specific case and say so in the table: a **patch** request where
the caller means "clear this field", as distinct from "leave it alone". If the API has no
patch semantics, `null` never appears.

Rules:

- The generated client's type marks an omitted-when-absent field optional, not nullable.
- The rehydration function distinguishes "absent" from "present and empty". A missing
  duration and a zero duration are different answers.
- An absent optional does **not** become a default during rehydration. Defaulting is a
  use-case decision, made where the meaning is known.

---

## 6. Rehydration in the client

This is the part that is usually left out, and it is where the value of the whole scheme
is either collected or lost.

**The generator emits, per contract type, a function that turns the parsed JSON into the
client's value types.** The transport core calls it once, on the response, before
anything else sees the payload. The symmetric function runs on the request on the way
out.

```ts
// generated — never hand-edited
export function rehydrateGetOrderResponse(raw: unknown): GetOrderResponse {
  const r = raw as RawGetOrderResponse;
  return {
    order: {
      orderId:   OrderId.parse(r.order.orderId),
      placedAt:  Instant.parse(r.order.placedAt),
      deliverOn: LocalDate.parse(r.order.deliverOn),
      slot:      r.order.slot === undefined ? undefined : Duration.ofSeconds(r.order.slot),
      total:     Money.of(r.order.total.amount, r.order.total.currency),
      status:    r.order.status,
    },
  };
}
```

Rules:

1. **Generated from the manifest, not written by hand.** A hand-written rehydrator stops
   tracking the contract the moment a field is added, and the failure is a silently
   missing value rather than a type error.
2. **Applied once, at the transport boundary.** Everything above it holds real values.
3. **No parsing in a component, a hook, a selector, a template, or a store action.** A
   date constructor call in a component is the symptom this whole file exists to remove.
4. **The client's own value types are the same library on web and mobile**, so one set of
   helpers serves both and a value crossing between them means the same thing.
5. **Rehydration failure is an error, not a silent default.** A field that will not parse
   is a contract violation; report it with the endpoint name and the field path. Do not
   substitute the current time, zero, or an empty string.
6. **Dehydration is generated too.** A request built by hand from strings reintroduces
   every problem on the way out.

### 6.1 What the manifest must carry for this to work

The extractor cannot emit a rehydrator if the manifest says a field is "a string". It
records the **semantic type**:

```json
{ "name": "deliverOn", "kind": "localDate", "optional": false }
{ "name": "slot",      "kind": "duration",  "optional": true  }
```

So the semantic type has to be visible to the extractor — from the declared type in a
language with real value types, or from an annotation where the underlying type is a
primitive. **Either way it is declared, never inferred from the field's name.** A rule
that treats any field ending in `At` as an instant is a guess that is wrong the first time
somebody writes `repeatAt` for a wall-clock time.

---

## 7. The specific mistakes this prevents

Each of these has a name, and each ships regularly in systems that carry dates as strings.

- **A date treated as an instant.** A delivery date of `2026-03-01` converted through a
  timezone becomes `2026-02-28T23:00:00Z` west of UTC, and the delivery moves a day. A
  date-only value has no zone and is never converted through one.
- **A wall-clock time treated as an instant.** An opening time of `09:00` is 09:00 in
  whatever zone the branch is in, on whatever day it is read. Pinning it to a date and a
  zone at write time is wrong for every subsequent day, and for the days a daylight-saving
  transition crosses.
- **Arithmetic across a daylight-saving boundary.** "Add 24 hours" and "the same time
  tomorrow" differ twice a year. The library distinguishes them; a string plus a number of
  milliseconds does not.
- **A duration as a formatted string.** `"1:30"` is ninety minutes to the writer and one
  and a half hours to the parser, and neither survives a locale change.
- **Money as a float.** `0.1 + 0.2` is the standard demonstration. Minor units, always.
- **A large integer as a JSON number.** Above 2^53 a browser silently rounds it. An
  identifier or an account number that is numeric is a string on the wire.
- **A client date type losing precision or being mutated.** Some platform date types are
  mutable and millisecond-resolution. Where the domain needs more, the client's value type
  is a real immutable type, not the platform primitive.
- **An enumeration as an ordinal.** Reordering the enumeration silently reassigns every
  stored value.

---

## 8. The store format is a separate decision

The store format and the wire format are **independent**. They often coincide, and
treating that coincidence as a rule is how a wire change corrupts a table.

- **Write down both**, per semantic type, in the same table as § 2 if they differ.
- A store that can index a value natively should usually hold it natively — a sortable
  timestamp, a numeric amount.
- A store that cannot should hold the lexicographically sortable string form, so a range
  query still works.
- **Changing a store representation is a data migration**, with the dual-read window from
  `05-repositories-and-persistence` § 5.1. Changing a wire representation is a contract
  change. They are not the same event and they do not have to happen together.

---

## 9. Persisted values on a client

A client that stores server data locally — the offline store in `10-mobile` — persists the
**dehydrated** form, and rehydrates on read. Never persist a rich value type's in-memory
representation: it changes when the library is upgraded, and the store outlives the
upgrade.

Consequences:

- **A converter change is a breaking change for the persisted copy**, exactly as it is for
  the wire. It bumps the local schema version and the client discards and re-fetches.
- **Rehydration runs on every read from the local store**, so it has to be cheap. It is
  also on a repeating path, which makes it subject to `10-mobile` § 5 — do not rehydrate
  a whole collection inside a render.
- **A value written offline and synced later carries the instant it was captured**, not
  the instant it was uploaded. That is a domain decision and it needs a field for each,
  not one field that means different things depending on the path.

---

## 10. Testing

| Test | Where | Asserts |
|---|---|---|
| Round trip, per converter | unit, beside the converter | `from(to(x)) == x` across the type's edges: leap day, daylight-saving transition, zero, minimum, maximum |
| Wire form, per converter | unit | the exact string or number produced, pinned. This is the contract; it does not change silently |
| Rehydrator, per contract type | generated code, exercised by a client unit test | every field arrives as the right type; an absent optional stays absent |
| Store round trip | integration | the mapper writes and reads back an equal value through the real store |
| Manifest coverage | a structural guard | every semantic type appearing in the manifest has a converter on the server **and** a rehydrator on each client. A new type cannot be added silently |
| Determinism | all unit tests | a fixed clock and a fixed identifier generator, so a timestamp assertion is a real assertion |

The manifest-coverage guard is the load-bearing one. Without it, the first field of a new
semantic type reaches the client as an unconverted string, and the failure is a value that
looks nearly right.

---

## 11. If the project chose Strategy A

Keep, from this file:

- § 2, the wire-format table. It is still a specification even when both sides hold
  strings, and the money, decimal, large-integer, optional and enumeration rows are
  unchanged.
- § 7, the list of mistakes. Strings do not prevent any of them; they only remove the
  compiler's ability to catch them, so the rules become review discipline instead.
- § 8, the store-format separation.
- The determinism row of § 10.

Drop the converter registry, the rehydration generation, and the manifest-coverage guard —
there is nothing to convert.

**And revisit the choice the first time the product grows recurrence, availability, or
multi-timezone users.** That is the point at which Strategy A stops being the cheaper
option, and moving later costs a pass over every date field in the system.
