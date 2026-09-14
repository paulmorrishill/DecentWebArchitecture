# 17 — Observability and error handling

The failure mode this file exists to prevent is not a crash. It is a system that keeps
running while doing nothing, and tells nobody.

---

## 1. One reporter, everywhere

Every package exposes the same function:

```ts
reportError(tag: string, error: unknown, context?: Record<string, unknown>): void;
```

- `tag` is a dotted path naming the site: `orders.cancel.publishFailed`. It is what you
  search for six months later, so it names the *site*, not the error.
- `context` carries identifiers and state — never personal data, never a credential.
- Behind it: a monitoring service in deployed environments, the console in local
  development. One implementation per package, chosen at composition.

---

## 2. Silent recovery is banned

Every catch block either rethrows or reports.

Banned patterns, in any language:

- an empty catch,
- a catch that declares the error and ignores it,
- a rejected-promise handler returning null, undefined, or an empty object,
- a catch that returns a default value with no log line.

**Recovery is fine. Silent recovery is not.** The correct shape is:

```ts
.catch((err) => { reportError('orders.pricing.failed', err, { orderId }); return null; })
```

Best-effort cleanups — temporary file deletes, teardown operations — still get a log
line. If one fails a hundred times a day you want to know.

### 2.1 Calls that succeed on failure

Some calls resolve successfully on a non-2xx response, or report partial failure in a
field rather than by throwing. A download helper that ignores the status writes the error
body to disk: the work "succeeds", the queue drains, the screen renders a broken asset,
and no log anywhere says why.

- Check the status after any call that can resolve on a failure.
- Throw outside the success range.
- Validate the post-condition where it is cheap: file size on a download, row count on a
  bulk insert, the status field on a batch operation.

Garbage written to disk is the failure mode a reporter alone cannot catch.

---

## 3. Work that never happened

No error exists, because nothing ran. A search for catch blocks cannot see any of this.

### 3.1 The detached call

A call whose result is discarded loses its rejection. Marking it as deliberately ignored
is worse: it silences the linter's floating-promise rule and takes the rejection with
it.

**Rule:** `await` it, or attach a reporting handler.

### 3.2 The frozen invocation

On a serverless host, work still pending when the handler returns is suspended, not
failed. A reporting handler never fires, because nothing rejected.

**Rule:** on a request path, background work is awaited before the response returns, or
handed to something durable — a queue, or an outbox row. See
[07-composition-and-entrypoints](07-composition-and-entrypoints.md) § 5.

### 3.3 The unwired collaborator

Optional-chaining a call to an injected collaborator converts "this was never wired" into
"did nothing, reported nothing".

**Rule:** a collaborator the use case needs is a **required** constructor parameter, so a
missing wiring fails loudly at startup. If absence is genuinely a designed state, the
absent branch logs, or a test proves the absent case is intended — so the next reader can
tell design from accident.

### 3.4 The no-op selected by configuration

```ts
config.pushTopicArn ? new RealBroadcaster(...) : new NoOpBroadcaster()
```

A no-op satisfies the interface and discards everything. An environment whose
configuration is empty is then indistinguishable from a working one, and the feature is
silently absent until a human happens to notice.

**Rule:** log which implementation was selected, at startup. One line in the first log of
every deploy.

### 3.5 The invariant proved in the wrong layer

A guarantee proved in an inner layer does not hold at the outer layer that wraps it. A
use case that "always writes a terminal status" says nothing about a handler that can
throw before the use case runs.

**Rule:** verify the invariant at the **outermost** layer that can violate it, reading
every early return and pre-call path in the wrapper — not only the inner function that
carries the comment.

---

## 4. Work that half happened

The tool ran, returned success, and did less than it was asked. Nothing to catch, nothing
missing to grep — only output that is quietly incomplete.

- After editing a build configuration or an entry map, **list the entries** and confirm
  the ones you did not mean to change are still there. A replace-instead-of-add drops a
  bundle with no failure: the deploy succeeds and the old code keeps running.
- After a test run, check the file and test counts against an expected total.
- After codegen, **open the output and find the thing you added**.
- A passing suite does not typecheck. Run the build.
- Read first-attempt failures separately from final counts.

---

## 5. Logging

- **Structured, not prose.** One event per line with named fields.
- **Startup logs the selected implementations**, the configuration that is present and
  absent (by name, never by value), and the build identifier.
- **`console.*` is not observability.** It is invisible in production and it is a swallow
  by another name. New code does not add one.
- **Never log a credential, a token, a full request body, or personal data.** Log
  identifiers.
- Set a retention period on every log destination.

---

## 6. The error taxonomy on the wire

| Kind | Status | Body | Reported? |
|---|---|---|---|
| Validation | 400 | code + user-readable message | no |
| Unauthenticated | 401 | code | no |
| Forbidden | 403 | code | no |
| Not found | 404 | code | no |
| Conflict | 409 | code + what to do instead | no |
| Unhandled | 500 | request id only | yes, with full detail |

Rules:

- **Each distinct failure gets its own code and message.** A single lumped error on the
  server guarantees a single lumped message in the client, and a test assertion that more
  than one path satisfies.
- A collision on a name someone else has taken is a **409 with an actionable message** —
  never a 403. A 403 says "you are not allowed"; the truth is that a different name
  would work.
- A 500's body carries the request id and nothing else. The detail is in the report.

---

## 7. Client-side

- A global error boundary catches render failures, reports them, and shows a screen with
  a way forward.
- Unhandled rejections and resource-load failures are reported too.
- A failed read renders a **visibly failed** state. Never an empty one — an empty list
  reads as "there is nothing here" and throws nothing, so no gate can see it.
- The user-facing message for each error branch is specific. A catch-all is
  indistinguishable from a swallowed bug.

---

## 8. Comments

- **Comments describe the code, not the change.** "Changed from X to Y", "now uses Z"
  means nothing to someone reading the file fresh in a year.
- **No ticket references in source.** Issue numbers belong in commit messages and pull
  request bodies, which carry their own context. A reference in a comment sends the
  reader to a tracker to learn nothing.
- A comment explaining *why* a non-obvious thing is done is valuable and should be
  written. A comment restating what the line does is noise.
- No dead code behind a comment. Delete it; the history has it.

---

## 9. Judging a measured number

A runtime wildly out of proportion to the work is a **defect**, not a statistic to report
beside a tick.

Rough anchors, not targets: an ordinary API request is sub-second to low seconds; a small
batch of a few dozen records is seconds, not minutes. A batch of twenty items taking
three minutes is eight seconds per item, and that is broken.

When you see one, do not call the work done. Find the cause — a per-item external call
with no batching, a query per row, a synchronous wait inside a loop, a real timeout being
retried, work inside a transaction — and fix it, or report it prominently as a
performance defect with the number and the diagnosis.

This is the opposite face of never predicting durations: **predicting** how long
something will take is worthless, and **judging** how long something took is required.
