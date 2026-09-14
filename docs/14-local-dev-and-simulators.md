# 14 — Local development and simulators

**The whole stack runs on a developer machine with no cloud account, no credentials, and
no network.** This is not a convenience. It is what makes the end-to-end suite runnable
in CI, on a laptop, and offline, and it is what keeps the feedback loop in seconds.

---

## 1. The local host

A small HTTP server that is a **fourth entrypoint with the same composition** as the
deployed one: same use cases, same repositories, same services — only the clients point
at local simulators.

It:

- boots every simulator on its own port,
- creates the store's tables from the **same definitions** the deployed environment uses,
- registers one route per endpoint from the **generated route map**, so an endpoint
  cannot exist in the deployed API and be missing locally,
- exposes a **reset** endpoint that wipes every simulator's state,
- exposes a mail viewer and any other inspection surface a developer needs.

One command starts the API, the web client, and the mobile web target together, with
prefixed and coloured output per process.

---

## 2. Simulators

One per external dependency. Each is a small in-process server implementing only the
operations the application actually uses.

| Dependency | Local stand-in |
|---|---|
| Document store | In-process emulator, tables created from the shared definitions |
| Relational store | A container, version-pinned, created and migrated per run |
| Object storage | A file-backed server implementing put/get/delete and signed URLs |
| Mail | A capture server: stores messages, exposes a list and a fetch endpoint |
| Identity provider | An emulator implementing the authorization-code flow with fixed users |
| Push channel | A local socket server |
| Model/API providers | A deterministic simulator returning canned, shaped responses |
| Queue | Direct in-process dispatch, or a simple queue server |

### 2.1 Rules

1. **Register every simulator with one reset registry.** A new simulator adds itself;
   the reset endpoint iterates the registry. A simulator that resets only when someone
   remembers to add it to a list is a source of cross-test contamination.
2. **Fixed users, fixed identifiers.** The local identity emulator offers one user per
   role with stable identifiers, so tests can assert on them.
3. **A simulator implements only what is used** — and fails loudly on anything else,
   rather than returning an empty success.
4. **Write down what each simulator fakes.** Every faked behaviour is a class of defect
   the suite structurally cannot find. Put that list in the local host's header comment
   and in the architecture guide.
5. **Never read a green suite as evidence about a component the local host replaces.** A
   double cannot fail to be wired.

### 2.2 Fault injection

A small module that makes a simulator fail on demand — a status code, a timeout, a
truncated body — reachable from a test. This is how the error paths in
[13-testing-e2e](13-testing-e2e.md) § 1.1c are exercised without a mock.

Use **non-retryable** failure types where the client SDK retries on its own, and assert
that the number of failures observed equals the number injected. A retryable type is
retried away and the harness reports a plausible smaller number.

---

## 3. Configuration

- One script generates the local configuration files from a template on first run, so a
  fresh clone starts with one command.
- Secrets required locally are placeholders. A build that requires a real credential to
  start is a build nobody can run.
- Where a real credential is genuinely needed for one feature, the startup check names
  the variable and what the feature is, and the rest of the application still runs.

---

## 4. Seeding

Two kinds, kept apart:

**Test seeding** — through real API endpoints, from the test's own helper. This is what
end-to-end tests use.

**Development seeding** — a script that populates a realistic dataset for a human
clicking around. It calls the same endpoints. It is never imported by a test, because a
shared seed makes every test depend on a fixture nobody owns.

---

## 5. Ports

Fix the ports and write them down. A table in the repository readme:

| Process | Port |
|---|---|
| API | 3001 |
| Web client | 3000 |
| Mobile web | 3002 |
| Store emulator | 3800 |
| Object storage | 3801 |
| Mail | 3802 |
| Identity | 3803 |

Rules:

- The end-to-end configuration starts its own servers on these ports and **never reuses
  a running one**. If a port is occupied, stop and report which process holds it.
- Never guess whether the thing on the port is yours. Check the process id and its
  uptime against when you last built. A health check proves something is listening, not
  that it is what you built.

---

## 6. The developer loop

```
<start>            everything, watch mode
<test:unit>        fast suite, seconds
<test:integration> store-backed suite
<test>             both
<e2e> <file>       one end-to-end spec
<build>            typecheck + generate + bundle, every package
<lint>
```

Before opening a pull request:

- [ ] Build clean across every package.
- [ ] Targeted unit tests pass for the code touched.
- [ ] Typecheck passes. **Use the project's real typecheck command** — a
      partially-configured one can check nothing and report success.
- [ ] Lint clean.
- [ ] Generation ran, and reported no contract violation.
- [ ] No hand-edits to any generated file. They are overwritten on the next build.

---

## 7. Testing against a deployed environment

Sometimes a defect only appears in a deployed environment. Two rules:

1. **An agent cannot type a credential into a sign-in form, and must not be asked to.**
   Provide a session tool that opens a real browser, drives the application's own sign-in
   link, supplies credentials from a local secrets directory, and leaves the browser open
   with a debugging port exposed. The agent drives that browser through the port.
2. **Never write a credential into browser storage to skip the sign-in interface.** A run
   that starts after the sign-in screen cannot find a defect in it.

Point such a tool at the **pre-production** environment only, and make that a property of
the tool rather than a flag someone must remember.

### 7.1 A health probe is an API call

To check whether an environment is up, call a real, cheap, unauthenticated API endpoint
on the **API host**. A path under the application host usually returns the client shell
with a 200 whatever the API is doing, so it is never a health signal.
