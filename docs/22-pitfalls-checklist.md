# 22 — Pitfalls checklist

Copy this into the new repository as `PITFALLS.md`. **Append a section every time a class
of defect reaches a deployed environment.** A checklist that never grows is a checklist
nobody is learning from.

Walk the relevant sections before opening a pull request, before claiming a task is
complete, and before merging.

---

## 1. New permissions required?

- [ ] Every cloud API the new code calls has a matching permission statement in the
      deployable's role.
- [ ] The permission names the specific action and the specific resource, not a wildcard.
- [ ] Check the attached-policy quota if you added a new policy. Prefer the inline form.
- [ ] Avoid a detach and attach in one apply — the tool may parallelise them and the
      quota check races.

## 2. New data stores created?

- [ ] Declared in infrastructure, not created by hand.
- [ ] The local environment creates the same shape from the same definitions.
- [ ] Key shape decided explicitly and stated in the pull request
      ([05-repositories-and-persistence](05-repositories-and-persistence.md) § 4).
- [ ] If tenant-scoped: the tenant attribute and tenant index exist in the infrastructure,
      and an isolation fixture exists.
- [ ] The deployable's role has read and write permission on it.

## 3. Build and typecheck after a backend change

- [ ] The full build ran, not just the test suite. **A passing suite does not typecheck.**
- [ ] The expected bundle or artifact exists on disk.
- [ ] Generation ran and reported no contract violation. No generated file appears in
      the diff.

## 4. Data leaking to a lower-privileged role

- [ ] Every response a lower-privileged role can reach has been read field by field.
- [ ] A response shaped for an administrator is not reused by a lower-privileged screen.
      It leaks in the network tab whether or not anything renders it.
- [ ] The privacy classification on every new or changed endpoint is present and correct.
- [ ] Permission tests exist both ways: usable for a permitted role, refused for an
      unpermitted one.
- [ ] No authorization decision reads the client instance identifier, a request field, or
      any other client-supplied value. Identity and roles come from the verified token
      only.

## 4b. Ambient state

- [ ] `execute` takes one argument. No context object reappeared, under any name.
- [ ] Each new use case declares the ambient-state ports it reads, individually, in its
      constructor. No bundle, resolver, locator, or factory is injected in their place.
- [ ] No use case reads a cookie, a header, or any other transport detail.
- [ ] No ambient clock call anywhere in the application layer.
- [ ] A use case taking more than about three ambient-state ports was looked at again.

## 5. Infrastructure

- [ ] Plan read line by line. No unintended replacement.
- [ ] No stateful resource in the destroy list.
- [ ] Resource names within the service's length limit.
- [ ] No cycle between a resource and its dependent.
- [ ] Any guard test over infrastructure parses its subject rather than quoting a literal
      line.
- [ ] Backend unit tests run, even for an infrastructure-only change, because guards live
      there.

---

## 6. Silent exception swallowing

A catch that returns a default without logging keeps shipping while the underlying failure
stays invisible — until something breaks downstream weeks later with no line to trace it
back to.

- [ ] Every new catch block rethrows or reports.
- [ ] No empty catch, no declared-and-unused error, no rejected-promise handler returning
      null.
- [ ] Any call that can resolve on a failure has a status check after it.
- [ ] Where the operation has a known expected size or checksum, it is validated after the
      write.

## 6b. Silent no-ops: work that never happened

No error exists, because nothing ran. A search for catch blocks cannot see any of this.

- [ ] No detached call that drops a rejection. Await it, or attach a reporting handler.
- [ ] Every fire-and-forget on a **request path of a serverless host** is awaited before
      the response returns, or handed to a queue or an outbox. A handler reports a
      rejection; it does not make suspended work run.
- [ ] No optional-chained call to an injected collaborator. A needed collaborator is a
      required constructor parameter.
- [ ] Every no-op implementation selected by configuration logs which one was chosen, at
      startup.
- [ ] Every best-effort side effect reports its own failure.
- [ ] The invariant is verified at the **outermost** layer that can violate it.
- [ ] Do not read a green end-to-end suite as evidence that any of this works. A double
      cannot fail to be wired.

## 6c. Partial success: work that half happened

The tool ran, returned success, and did less than it was asked.

- [ ] Build-config edits: the entries you did not mean to change are still listed.
- [ ] Suite runs: file count and test count match the expected total, not just exit zero.
      A file you added shows its own line.
- [ ] Baseline counts measured on **this branch's own merge base**, never remembered from
      elsewhere.
- [ ] The build passed, not only the suite.
- [ ] Codegen output greps clean for the symbol you added.
- [ ] Security-relevant artifacts (the route table, the privacy table) are generated by
      the deploy from the deployed revision, and a generation failure fails the deploy.
      No cached or previously built copy is ever used.
- [ ] Browser-suite runs: first-attempt failures read separately from the final counts. A
      retry-pass is triaged, not accepted.

## 6d. A green test that cannot fail

- [ ] Every new assertion has been seen to **fail** — by breaking the code — not only to
      pass.
- [ ] No assertion is satisfied by more than one outcome of the code under test.
- [ ] No test compares the code under test to a second copy of its own logic.
- [ ] Any enumerated list of things to check is derived from a growing source, or guarded
      by a count against one.
- [ ] A test double enforces the constraint the real thing enforces, or the test says what
      it is scoped to.

## 6e. Doubles and error assertions that pass for the wrong reason

- [ ] Each lookup double answers only for the identifier it holds, proven by mutating the
      production lookup and watching the test go red.
- [ ] Multi-identifier doubles proven **per identifier**, one argument at a time.
- [ ] The load-bearing assertion is an allow or write case, not only denials.
- [ ] "Never consulted" is proven with a **throw-on-call** double; "consulted but
      immaterial" with an identifier-aware double plus a mutation that still passes. The
      two are labelled distinctly.
- [ ] No bare error-class assertion where several errors of that class reach the call.
      Assert the message.

## 6f/6g. The instrument lied

Any time a number is about to become evidence:

- [ ] The mutation is confirmed present in the tree before any after-count is read. An
      identical after-count is the signature of "did not apply", not "did not bite".
- [ ] The restore is confirmed to have reached the file — read the diff, not the exit
      code. A restore can revert an uncommitted edit of your own along with the mutation.
- [ ] A two-state comparison separates two **actual** states. Reverting an
      already-committed change removes nothing.
- [ ] The count's magnitude matches the claimed scope — the right file argument was
      passed.
- [ ] Any process under measurement is the one just built, checked by process id and
      uptime, not by a health check.
- [ ] A hang with no error is not a synchronous child racing a server in its own event
      loop.
- [ ] A sweep searched every call shape and helper name, not one field name.
- [ ] A counter-example failed the way the hypothesis predicted — names and error read,
      not just the total.
- [ ] Every mutation count is reported with its patch width.
- [ ] A composite-key mutation pins a part to a constant resolving a **seeded neighbour**,
      and the result read is the dimension's own named test.
- [ ] A failure-injection harness uses non-retryable error types and asserts failures
      observed equals failures injected.
- [ ] A process is stopped by id where you hold one, never by a pattern that can match
      your own command line.

---

## 7. Refactors

- [ ] Blast radius mapped before editing: every handler, client screen, mobile screen,
      tool, scheduled job, and infrastructure module that names the thing.
- [ ] Every contract type crossing the boundary listed.
- [ ] Store changes reviewed for destroy-and-recreate.
- [ ] Contract changes keep old clients working for one release.
- [ ] Client-side offline stores considered: an old app holds the old shape.
- [ ] No type suppression, no widening, no dropped log line "to be fixed next pass".
- [ ] Full gate run, **including** the end-to-end step.
- [ ] The pull request lists what was verified and the compatibility window for any client.

---

## 8. Tenancy

- [ ] New scoped table has its tenant attribute and index, and an isolation fixture.
- [ ] New anonymous endpoint touching scoped data has a registered tenant resolver.
- [ ] New worker resolves the tenant from the row it processes.
- [ ] New push payload carries the tenant.
- [ ] No repository subclass reaching past the base class to the raw client.
- [ ] New static asset path resolves the tenant directory through the helper, never a
      literal.
- [ ] Key shape stated, and a same-name-in-two-tenants test exists for any caller-chosen
      key.

---

## 9. The write succeeded and the user cannot see it

A create, delete, or rename that updates the server and leaves the screen unchanged reads
to the user as a failure. They retry, and now there are two.

This survives to production because the obvious test passes: a test that stops at "the
call returned success" asserts the half that worked and never looks at the list.

- [ ] For each create, delete, or rename in the diff, name the collection it belongs to
      and the line that refreshes it.
- [ ] The collection refreshed is the one that **receives** the item, not a sibling.
- [ ] If the item lands in a collapsed or paginated container, it is revealed.
- [ ] The end-to-end test asserts the new row is **rendered**, by test id.
- [ ] A delete test asserts the row is **gone from the DOM**.

---

## 10. Collapsing a set that can hold more than one

Taking the first element, or a limit of one, from a collection whose real cardinality is
greater than one is silently wrong: no error, and the caller gets one arbitrary element
or nothing.

The general shape: **a limit or a first-row read applied before a filter is applied to the
wrong set.**

- [ ] Any limit on a post-filtered query: state which set it limits, pre- or post-filter.
- [ ] A first-element read on a multi-row result carries either a comment stating why the
      domain guarantees one, or code that handles all of them.
- [ ] Uniqueness enforced at write time within a scope does not make a wider index
      single-valued.

---

## 11. Polling a healthy endpoint, hammering a failing one

- [ ] No timer-driven re-fetch of server state in any client.
- [ ] Every reconnect and retry names its cap and its backoff in the diff.
- [ ] Any timer that re-arms itself on failure is treated as a retry loop.
- [ ] On exhaustion it is surfaced and reported, not abandoned silently.

---

## 12. Cross-language guards

- [ ] A new guard over configuration or infrastructure **parses** its subject; it does not
      assert an exact formatted line.
- [ ] A pull request touching infrastructure or pipeline configuration records that the
      backend unit tests were run.

---

## 13. Interleaved-append conflicts

- [ ] Any conflict in an append-only region — an index table, a test block, a registry
      array — was resolved by keeping **both** additions intact, not by editing across the
      markers.
- [ ] A sequence number taken for a new record was checked against every pushed branch,
      not only the default branch and not only open pull requests.
- [ ] A claim about what a commit contains quotes the file list, never the subject line.

---

## 14. Layout assertions

- [ ] Measured against the element's **offset parent**, not the viewport and not the
      document. Inside a horizontally scrolling container both of those measure the
      container.
- [ ] The assertion has been seen to fail with the fix removed.

---

## 15. A control every test opens and none closes

- [ ] Both directions of every two-state control are covered.
- [ ] The instances are enumerated, not sampled.
- [ ] The second state is asserted **after** the render that follows it, and again after
      provoking an unrelated re-render.
- [ ] A route-based default lives in a state change, not in the rendered expression.
- [ ] No existing test depends on the defect — a blind click on a control whose state was
      never established is a toggle, not an open.

---

## 16. A feature that is a link to a third party

- [ ] Someone followed the link once against the real destination and confirmed the page
      exists and does what the feature claims.
- [ ] If the address is hand-built, a unit test pins the parameters the far side requires.
- [ ] The pull request says who took the manual pass and what they saw.

---

## 17. Work that repeats on a mobile client

- [ ] Anything repeating is gated on **application-active** state, not navigation focus.
- [ ] No serialisation of a growing payload inside a render or a per-event handler.
- [ ] No store subscription without a selector.
- [ ] No scan or sort of an unbounded array per event.
- [ ] No read-modify-write of a whole record per event.
- [ ] No second writer for data a background task already records.
- [ ] For the main repeating path: one real-device run, screen locked, reporting mean CPU
      and confirming no suspension line in the system log.

---

## 18. Imported general-quality rules

Shorter rules with no incident behind them yet.

- [ ] Comments describe the code, not the change.
- [ ] No ticket references in source.
- [ ] Identifiers spelled out. No dropped vowels.
- [ ] No `(s)` pluralisation in user-facing text. Format on the count.
- [ ] No new `console.*` — it is invisible in production and is a swallow by another name.
- [ ] No development-only shim that changes shipped behaviour on an authentication, sync,
      or network path.
- [ ] Every user-initiated server call shows it is working and blocks a second trigger.
- [ ] File bytes in object storage; the row holds a key.
- [ ] No hand-declared duplicate of a generated client type.

---

## When to run which section

| Situation | Sections |
|---|---|
| Any backend pull request | 1–6c, 8 |
| Any pull request adding or editing a test | 6d, 6e |
| Any reported measurement | 6f/6g |
| A refactor | 7, plus everything |
| A client pull request | 9, 10, 11, 14, 15, 18 |
| A mobile pull request | 17, plus 7 for contract compatibility |
| Infrastructure or pipeline only | 5, 12 |
| A merge or rebase conflict in an append-only region | 13 |
| A feature that links to an external console | 16 |
| Before merging to the integration branch | all applicable |
