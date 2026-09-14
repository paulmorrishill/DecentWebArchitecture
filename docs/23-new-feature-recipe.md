# 23 — The new-feature recipe

End to end, from a rough request to a merged change. Each step links to the file that
covers it.

---

## Phase 0 — Before any code

### 0.1 Settle the working tree

Fetch, measure, ask which working tree to use, settle uncommitted changes. Wait for the
answer. ([21-agent-working-rules](21-agent-working-rules.md) § 3.1)

### 0.2 Harden the specification

A feature described in a sentence is not ready to build. Answer these before writing
anything, and take the ones only the owner can answer back to them:

- [ ] **Problem.** What is wrong today, for whom?
- [ ] **Scope.** What is explicitly out?
- [ ] **Acceptance criteria.** What observable behaviour proves it is done?
- [ ] **Data.** What rows are created? What is the key shape, and where does the key value
      come from? ([05-repositories-and-persistence](05-repositories-and-persistence.md) § 4)
- [ ] **Tenancy.** Scoped or global? If global, why, and what stops one tenant reading
      another's data? ([08-multi-tenancy](08-multi-tenancy.md))
- [ ] **Roles.** Who may do each operation? What does each role see?
- [ ] **Personal data.** Does any response carry it? ([18-security-and-privacy](18-security-and-privacy.md) § 3)
- [ ] **Deletion.** What happens to related data? Soft or hard? Who may?
- [ ] **Scale.** Does any collection grow without bound? Pagination decided?
      ([19-performance-and-scale](19-performance-and-scale.md) § 1)
- [ ] **Notification.** Does anything need to tell a user? By what channel?
- [ ] **Clients.** Which of web, mobile, and machine-facing tools are affected?
- [ ] **Compatibility.** Does this change an existing contract? What is the overlap
      window? ([04-use-cases](04-use-cases.md) § 7)
- [ ] **Test plan.** Which suites cover which parts, and what genuinely cannot be covered
      end to end — listed explicitly, for the owner to confirm.

### 0.3 Record the decision if there is one

If the feature makes a choice that is hard to reverse, non-obvious, or externally
constrained, write the ADR **in this pull request**. ([20-decision-records](20-decision-records.md))

---

## Phase 1 — The backend slice

1. **Domain entity.** Add or extend the type in `domain/entities/`.
   ([03-backend-domain-and-ports](03-backend-domain-and-ports.md) § 1)
2. **Port.** Add the methods to the relevant port, named as domain verbs. One port per
   aggregate. (§ 2)
3. **Contracts.** Write `XxxRequest` and `XxxResponse` in the use-case file, with named
   nested types. Nothing reused from another endpoint. ([04-use-cases](04-use-cases.md) § 4)
4. **Use case.** Decorator with namespace, method, roles, and the privacy flag. Class,
   `getRequiredRoles()`, `execute()`. Validate, check existence, check ownership, apply
   business rules, mutate, map, return. ([04-use-cases](04-use-cases.md) § 3)
5. **Unit test** beside the use case, with inline fakes. Happy path, every validation
   failure, not-found, permission denial, downstream failure. Make each new assertion
   fail before trusting it. ([11-testing-unit](11-testing-unit.md))
6. **Repository implementation.** Mappers module-local, table names as constructor
   parameters, conditional writes where concurrency matters.
   ([05-repositories-and-persistence](05-repositories-and-persistence.md))
7. **In-memory fake** updated to match the port, enforcing the same constraints.
8. **Integration test** for the new store behaviour: mapper round-trip, condition
   expressions, index queries, pagination.
   ([12-testing-integration-and-guards](12-testing-integration-and-guards.md) § 1)
9. **Wire it** into the namespace index and the composition root.
10. **Regenerate** contracts and clients. Fix violations until the generator exits zero.
    **Open the output and find the new endpoint.** ([06-api-contract-and-codegen](06-api-contract-and-codegen.md))
11. **Typecheck and build.** A green suite does not typecheck.

---

## Phase 2 — Infrastructure, if the feature needs it

1. Declare new stores, queues, buckets, or permissions in the relevant module.
2. Add the tenant attribute and index if the table is scoped.
3. Mirror the table definition into the local environment's definitions — same source
   where possible.
4. Read the plan. No unintended replacement, nothing stateful in the destroy list.
   ([15-infrastructure-and-environments](15-infrastructure-and-environments.md) § 4)
5. Run the backend unit tests: structural guards live there and an infrastructure-only
   change can break them. ([12-testing-integration-and-guards](12-testing-integration-and-guards.md) § 2.3)

---

## Phase 3 — The web client

1. **Call the generated client** from a hook or a domain service — never from a component.
   ([09-frontend](09-frontend.md) § 2)
2. **Render four states**: loading, empty, populated, failed. A failed read is visibly
   failed, never empty. (§ 6)
3. **After a mutation, refresh the collection that receives the item.** (§ 3.1)
4. **A control that writes disables itself for the duration**, with the guard at the top
   of the handler. (§ 5)
5. **Handle every error branch** with a specific, actionable message.
6. **Add test identifiers** to every element a test will touch. (§ 8)
7. **Component tests**: logic tests for the model, a rendering test for anything the
   template decides. (§ 9)

---

## Phase 4 — The mobile client, if affected

1. Regenerate the client. Never hand-edit it.
2. Store action, selector-based subscription, persistence wired in the store.
3. Sync: payload version bumped if the shape changed incompatibly; no field type changed
   in place.
4. Downloads check their status and validate the post-condition.
5. **If anything was added to a repeating path**, walk the CPU checklist and take a real
   device measurement. ([10-mobile](10-mobile.md) § 5)

---

## Phase 5 — End-to-end

1. **Page object first**, under `tests/e2e/pages/`. Both directions of any two-state
   control. ([13-testing-e2e](13-testing-e2e.md) § 3)
2. **Seeding helper** if the prerequisites are tedious through the interface.
3. **Spec**: reset, authenticate, seed, act, assert on rendered state.
4. **Assert the create is rendered and the delete is gone.** (§ 6)
5. **Permission spec**: usable for a permitted role, refused for an unpermitted one.
6. **Tenancy spec** where the feature is scoped: created in A, absent in B.
7. **Volume spec** where the screen shows a growing collection. (§ 8)
8. **Run the new specs three times with retries disabled.** A retry-pass is a defect to
   triage.

---

## Phase 6 — The gate

- [ ] Build clean, every package.
- [ ] Unit tests green; new assertions seen to fail for the right reason.
- [ ] Integration tests green.
- [ ] Client component tests green.
- [ ] New end-to-end specs green, three runs, retries disabled.
- [ ] Generated artifacts regenerated; tree clean **after** the run.
- [ ] Infrastructure plan reviewed, if touched.
- [ ] [22-pitfalls-checklist](22-pitfalls-checklist.md) walked for every area touched.
- [ ] ADR included, if the feature made a decision worth recording.

---

## Phase 7 — The pull request

The body states, plainly:

- **What** changed and why, in the controlled English of [21-agent-working-rules](21-agent-working-rules.md) § 2.
- **The key shape** of any new scoped table and where its value comes from.
- **The compatibility window** for any contract change: which client version, how long.
- **What was verified**, by name: which suites, how many runs, what the counts were.
- **What was not verified**, and why.
- **The infrastructure plan summary**, if applicable: in-place updates only, no destroys.
- **Anything deliberately left out of scope**, on one line each.

Then confirm with the owner before pushing, before opening, and before merging. Each is
its own approval.

---

## Phase 8 — After the merge

- [ ] The integration branch's own checks are green for the merge commit.
- [ ] The pre-production deploy completed, and a real request to the API host returns a
      2xx. ([14-local-dev-and-simulators](14-local-dev-and-simulators.md) § 7.1)
- [ ] Where the feature is user-visible, one manual pass against the deployed environment,
      as a real signed-in user.
- [ ] Where a defect reached a deployed environment that the local suite did not catch,
      raise a follow-up to close that gap — and append a section to the pitfalls
      checklist.
