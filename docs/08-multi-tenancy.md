# 08 — Multi-tenancy and data isolation

Read this before the first table exists. Retrofitting tenancy is the most expensive
refactor in this architecture, and the defects it produces are data leaks.

If the product genuinely has exactly one tenant forever, skip the mechanism but keep
§ 6: the primary-key rules apply to any scoped namespace, tenanted or not.

---

## 1. The model

- Every tenant has an identifier: a short stable slug, not a display name.
- Most domain rows carry a `tenantId` attribute and are readable only within their
  tenant.
- Some data is deliberately **global** — shared reference data, a public namespace, a
  cross-tenant administrative record. Global is a decision that is written down, not a
  default.
- A user belongs to one tenant, or holds membership of several. Decide which, early,
  because it changes the key rules in § 6.

---

## 2. The scoping mechanism

Three parts:

1. **A tenant provider** resolved per request. The entrypoint builds it from the request
   context and passes it to the repositories that need it.
2. **A tenant-scoped repository base class** that every scoped repository extends. It
   owns the primitives: `save`, `saveNew`, `saveMany`, `findByKey`, `findAllForTenant`,
   `queryForTenant`, `deleteByKey`, `applyTypedUpdate`.
3. **An index on the tenant attribute** so "everything for this tenant" is a query, not
   a scan.

### 2.1 The base class is the boundary

**A scoped repository never reaches past the base class to the raw store client.** A
direct client call inside a subclass bypasses tenancy enforcement entirely, and nothing
in the type system objects. If an operation cannot be expressed with the primitives,
**extend the base class**. That keeps the enforcement in one reviewable place.

### 2.2 Reads are filtered, and the filter runs where it can

`findAllForTenant` uses the tenant index. `queryForTenant` may filter after the store
returns — in which case **any row limit passed into it limits the wrong set**. Never
pass a limit to a post-filtered query unless the key condition alone already guarantees
at most one row per tenant.

### 2.3 Writes are guarded

`save` checks the row's existing tenant before overwriting. A cross-tenant overwrite
raises a distinct ownership-violation error that never reaches a client as-is — it is a
programming error, reported and turned into a 404 or a 409 depending on what the caller
was asking for.

Bulk writes need per-item conditions to keep that guarantee. Do not replace a
transactional bulk write with a cheaper unconditional one to save cost; the safety
property is the point.

---

## 3. Resolving the tenant

| Caller | Source of the tenant |
|---|---|
| Authenticated user | A verified claim on the token |
| Privileged user acting elsewhere | A header, honoured **only after** the role check that permits it |
| Anonymous public endpoint | An explicit resolver registered for that endpoint |
| Worker | Read from the row or message being processed |
| Scheduled job | Iterate tenants explicitly; never assume one |

### 3.1 Anonymous endpoints need an explicit resolver

An endpoint reachable with no token cannot read a claim. Register a resolver per such
endpoint — from a subdomain, a path segment, or a lookup on the entity being addressed
(a public link's own record knows its tenant).

Where no resolver is registered, the entrypoint supplies a provider whose `get()`
**throws**. That is a safety net, not a fallback: an anonymous endpoint that touches only
global data should never call a scoped method anyway. Do not register a fake resolver to
silence a stray scoped call — fix the use case.

### 3.2 Workers derive it from the data

Parse the event → point-read the entity with the raw client → read its tenant → wrap in a
constant tenant provider → construct the scoped repositories. Every worker that touches
scoped data follows this shape. Skipping the lookup and defaulting silently mixes
tenants.

---

## 4. Push and realtime

Every broadcast carries the tenant on the envelope. The fan-out reads it and delivers
only to connections in that tenant. A payload with no tenant is dropped, loudly.

Two consequences to document in any feature that uses the channel:

- The connection's tenant is fixed at handshake from the token's claim. A privileged user
  browsing another tenant over HTTP will **not** receive that tenant's pushes.
- Every publish site must have a concrete tenant in scope: use cases take it from the
  `TenantContext` port, workers resolve it from the row. A cross-tenant broadcast is a bug.

**Echo suppression uses the client instance identifier, not the user identifier.** A
broadcast caused by one client should not be re-applied by that client, which already has
the change — but the same user may have two tabs or two devices open, and both of the
others do need it. Stamp the originating instance identifier
([03-backend-domain-and-ports](03-backend-domain-and-ports.md) § 3.3.2) on the envelope
and let each client ignore its own. Suppressing by user identifier drops the update on the
user's other windows.

---

## 5. Static assets are scoped only by their path

Files served from a public bucket or CDN with no authorization are separated by nothing
but the directory in their URL.

- Load every such file through one helper that resolves the active tenant's directory. A
  literal path in a component is a cross-tenant leak no repository guard can catch.
- **Anything person-level stays behind the API.** Do not publish it as a static asset;
  it is world-readable to anyone who guesses a directory name.
- A tenant may legitimately have no file for a given data source. Declare that in a
  registry and let a per-tenant manifest decide whether the feature is offered, rather
  than shipping something that returns a 404 when switched on.

---

## 6. Primary-key shape

**A scoped table whose primary key is a caller-supplied value is broken until you prove
otherwise.** Two tenants pick the same value, address the same row, and the second
write hits the ownership guard — which the user sees as a bare 403 with no hint that a
different name would work.

For every new or changed scoped table, state in the pull request where the key comes
from:

- [ ] **Generated identifier** — cannot collide. Note it and move on.
- [ ] **User-chosen name or slug** — must be tenant-prefixed at the repository boundary.
      Encode on write, decode on read, and keep the caller-facing identifier plain
      everywhere above the repository. Ship a test proving two tenants can use the same
      name.
- [ ] **User identity** — say deliberately whether that identity can repeat across
      tenants. If users belong to exactly one tenant, a bare identity key does not
      collide *today*; write that down rather than leaving it to inference, because the
      day membership becomes movable it silently starts colliding. If the identity
      genuinely follows the person across tenants, the table is global and must not
      extend the scoped base class.
- [ ] **Externally-owned identifier** — shared across tenants by definition. Must be
      prefixed.
- [ ] **Composite key where a second attribute separates tenants** — acceptable, listed
      explicitly in an allowlist, with the reason written down.

### 6.1 Deliberately global namespaces

Some namespaces are global by design: a public short link, a publicly resolvable slug
whose resolver has no tenant context. Where that is right:

1. Document the invariant — why global is correct and what stops one tenant reading
   another's data through it.
2. Give the collision its own error value on the affected endpoints — `SlugAlreadyTaken` —
   so the client can tell the user a different name would work. Never let an ownership
   violation reach the client.

### 6.2 Changing a key shape strands existing rows

Read both shapes during the transition — scoped first, bare second, and note that the
bare fallback must still mask other tenants. Ship a re-key script alongside. Delete the
fallback once the script reports zero remaining rows **in every environment**.

---

## 7. Tests that enforce isolation

Three, and they are cheap:

**1. A cross-tenant isolation fixture per scoped repository.** One test walks a fixture
list: create in tenant A, read/list/update/delete as tenant B, assert nothing leaks.
A **discovery sentinel** in the same test fails the build when a scoped repository has no
fixture — that is what stops the next table from being added without a test.

**2. An identity-key sentinel.** Walk the same fixtures, force every caller-identity
field to one value, and fail when a key attribute comes back as that bare identity for a
table not on the documented allowlist. **A failure is not a request to extend the
allowlist** — decide first whether the table should be prefixed.

**3. An infrastructure guard.** Assert that every scoped table declared in code has its
tenant attribute and tenant index in the infrastructure definitions. Without the index,
`findAllForTenant` throws at runtime in a deployed environment, and neither typecheck nor
a test with a fake repository sees it.

Plus, per feature:

**4. A leakage end-to-end test** where the feature is user-visible: create in tenant A,
sign in as tenant B, assert absence in the list, the search, and any client sync payload.

If a feature is intentionally global, say so in the design and skip test 4 — but say it,
in writing, in the same pull request.

---

## 8. Mixed-scope features

A feature can legitimately be partly global. Example shape: a public resolver reads by
primary key with no tenant context, while the administrative list, update, and delete are
tenant-scoped; a related table of events is written by the anonymous handler and is not
scoped, because every read of it is gated behind a scoped lookup of its parent.

That is a sound design and it is also exactly the kind of thing nobody can reconstruct
six months later. **Write the security invariant down** — which reads are gated by what
— in an ADR, in the same pull request.
