# 19 — Performance and scale

---

## 1. Pagination is decided when the endpoint is written

Any endpoint returning a collection that grows with usage is paginated from its first
version. "It is only ever a handful of rows" describes the seed data, not production.

- The request carries a page and size, or a cursor. A cursor is better on a document
  store, where skipping rows costs the same as reading them.
- The response carries the next cursor, or the total where the store can give one
  cheaply.
- The client **pages or virtualises**. A paginated endpoint feeding a client that fetches
  every page in a loop has achieved nothing.

**Tests required:** a test that drives the pagination controls — forward, back, page size
— and a test seeded with **hundreds of rows** proving the screen stays correct and
responsive.

An un-paginated list is correct on a five-row seed and falls over once real data
accumulates.

---

## 2. The query-per-row shape

The most common latency defect: a list read followed by a lookup per item.

Symptoms in a diff:

- an `await` inside a loop over a collection,
- a repository call inside a mapper,
- a display-name resolution per row,
- a permission check per row.

Fixes, in order of preference:

1. Fetch the related set in one batched read and join in memory.
2. Denormalise the field onto the row at write time, where it is stable.
3. Read through a short-lived in-process cache for the duration of one request.

**Read a collection endpoint's implementation for this shape before merging it.** It does
not show up in a unit test, because the fake answers instantly.

---

## 3. Judge measured latency

A runtime wildly out of proportion to the work is a **defect**, not a statistic to report
beside a tick.

Anchors, as rules of thumb and not targets:

| Work | Expected order |
|---|---|
| A single API request | sub-second to low seconds |
| A batch of a few dozen records | seconds |
| A page of a list | sub-second |
| A full-suite end-to-end run | tens of minutes, and worth watching |

A batch of twenty items taking three minutes is eight seconds per item. That is broken.
Do not call it done. Find the cause — a per-item external call with no batching or
parallelism, a query per row, a synchronous wait inside a loop, a real timeout being
retried, work inside a transaction — and fix it, or report it prominently as a
performance defect with the number and the diagnosis.

**"It completed" is not "it is acceptable."**

---

## 4. Retries, backoff, and caps

Every error-triggered re-call has **a maximum attempt count or an overall deadline, and a
growing delay between attempts.** Both, not either.

- A server outage plus an unbounded client retry is self-inflicted load that also stops
  the outage clearing.
- On exhaustion, surface it and report it. Giving up silently is a swallow.
- A queue item that has exhausted its attempts is skipped and reported, not replayed on
  every drain.
- A reconnecting socket counts its consecutive failures and stops, rather than
  reconnecting forever.
- Any timer that re-arms itself on failure **is** a retry loop. Treat it as one.

---

## 5. No polling

There is a push channel for server state. A poll added because push "seemed flaky" hides
the broken push path, and the hidden breakage then sits there indefinitely. Lengthening
the interval does not make it acceptable.

- No timer-driven re-fetch of server state in any client.
- A timer driving *local* state — a clock, a recording duration, a countdown, an
  animation — is fine and is not what this bans.
- When you grep for timers, expect to dismiss most hits. The grep is a starting point,
  not a verdict.

---

## 6. Caching

| Layer | What | Invalidation |
|---|---|---|
| CDN | fingerprinted assets | never; the entry document is no-cache |
| Client memory | collections read by a screen | on mutation of that collection |
| Server memory | rarely-changing reference data | short time-to-live, per instance |
| Object storage | expensive derived payloads | rebuilt by a job; a time-to-live on the index |

Rules:

- **A cache without a stated invalidation rule is a bug with a delay.** Write down what
  invalidates each one.
- Client caches do not auto-evict on navigation. They revalidate on the next mutation.
- A server cache on a horizontally-scaled host is per instance. Do not rely on it for
  correctness.

---

## 7. Payload size

- Return what the screen needs. A list endpoint returning full nested detail per row is
  a latency defect and a privacy one.
- Where a screen needs a summary and a detail, that is two endpoints with two contracts.
- Large or binary payloads go through object storage with a time-limited URL, never
  through the API response body.
- Compress. Check it is actually happening.

---

## 8. Work on a repeating path

This is where mobile differs sharply from the web, and it is covered in full in
[10-mobile](10-mobile.md) § 5. The short form:

**Work added to a repeating path — a location fix, a timer, a socket message, a store
write, or a render caused by one of those — is measured on a real device before it counts
as done.** A mobile platform suspends a process that exceeds its background CPU budget,
reports nothing, and the only symptom is a hole in the data.

The same shape is a lesser problem on the web: a subscription without a selector, a
serialisation inside a render, a sort of an unbounded array per event. It costs battery
and jank rather than suspension, and it is still worth fixing.

---

## 9. Cost as a performance property

On a per-request billing model, an inefficient shape changes the bill, not just the
latency:

- A scan where a query would do.
- A transactional bulk write where a plain one gives the same guarantee.
- A log line per item in a large loop.
- An unbounded log retention.

Know which resources bill per request and which per hour, and wire a spend alert per
environment.
