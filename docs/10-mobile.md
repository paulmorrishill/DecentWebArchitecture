# 10 — Mobile application

The mobile app is a separate package with its own build, its own store, and two
constraints the web client does not have:

1. **Its local data outlives a server deploy.** A user may not update for weeks.
2. **The platform suspends it for using too much CPU in the background**, and reports
   nothing when it does.

Everything below follows from those two.

---

## 1. Structure

```
mobile/src/
  screens/            one component per screen
  components/         reusable UI
  navigation/
  store/              state stores, one per concern, each with its tests
  sync/               sync scheduler, queues, caches, diagnostics
  api/
    client.generated  generated from the same manifest as the web client
    client.ts         transport wrapper: credential, base URL, retries
  auth/
  hooks/
  tasks/              background task registrations
  media/
  config/
  types.ts
mobile/e2e/           device flow tests
mobile/scripts/       device harnesses
```

Conventions:

- Screens are PascalCase files; hooks are camelCase files named for the hook.
- Functional components with hooks. No class components in new code.
- The generated client is never hand-edited. Regenerate from the backend.

---

## 2. State

One store per concern, not one store for everything:

| Store | Holds |
|---|---|
| `dataStore` | the primary domain state and its sync status |
| `authStore` | credentials and identity |
| feature stores | one per bounded feature |

Rules:

1. **Components subscribe with a selector.** Subscribing to the whole store re-renders
   the component on every unrelated write — which on a repeating path is the CPU defect
   in § 5.
2. **Components dispatch; logic lives in store actions or pure helpers.** Same separation
   as the web client.
3. **Persistence is wired in the store, not in components.** Small values in key/value
   storage; structured data in the embedded database.
4. **A mutation notifies the sync layer through one function.** A screen never calls
   sync directly.
5. **Every store action that changes persisted shape is tested.** These files are where
   offline corruption comes from.

---

## 3. Sync

One scheduler owns every upload and download. It is idle until a mutation or a manual
trigger.

```
mutation → notify → debounce → sync run → (success) idle
                                        → (failure) backoff → retry
```

Parameters to choose and write down:

| Parameter | Purpose |
|---|---|
| Debounce window | Coalesce a burst of mutations into one run. A couple of seconds. |
| Backoff ladder | Growing delay on repeated failure, with a ceiling. |
| Chained-run cap | Maximum consecutive re-syncs without going through the debounce again. Guards against a pathological loop. |
| Network awareness | Offline defers and retries on reconnect. |
| App-state gate | Foreground/background changes gate eligibility. |

Rules:

1. **Conflict resolution is a stated rule, not an accident.** "Local unsynced mutations
   win over the server copy" is a fine rule; so is the opposite. Write down which, and
   test it.
2. **The download payload carries a schema version.** When the shape changes
   incompatibly, bump it and have the client discard and re-fetch rather than merge two
   shapes.
3. **Never change a sync field's type in place.** Add a new field; let the old one
   wither. The device holds rows written by an older app.
4. **Deleting a sync field needs an on-device migration path.** The local store still has
   rows carrying it.
5. **Large binary uploads go through a separate queue** from the data sync: a cache, a
   queue with per-item attempt counts, and direct upload to object storage through a
   time-limited URL.
6. **A queue item that has exhausted its attempts is skipped and reported**, not replayed
   on every drain.
7. **Deduplicate before upload** where the data allows it — successive location fixes
   within a few metres, identical consecutive states.
8. **A diagnostics snapshot** captures sync state for support. It carries no more
   personal data than it needs, and redacts precise location and contact details.

---

## 4. Downloads must check their status

A download helper that resolves "successfully" on a non-2xx response writes the error
body to disk. The queue drains to zero, the screen renders a broken asset, and no log
anywhere says why.

- Check the status field after every download returns, and throw outside the success
  range.
- Validate the post-condition where it is cheap: file size, expected content type,
  checksum.
- The same applies to any call that reports success by resolving rather than by status.

---

## 5. Work that repeats costs CPU, and CPU suspends the app

**Work added to a repeating path is measured on a real device before it counts as done.**

Mobile platforms enforce a background CPU budget. Exceed it and the process is
suspended; a location assertion is released; the app stops receiving updates until
something wakes it. **The app does not crash and reports nothing.** The only symptom is a
hole in the data.

The shape that produces it is never one obviously-wrong line. It is a screen that stays
mounted while the device is in a pocket, re-serialising a growing payload on every
render, driven by a timer, with an animation on top.

**Check every change that adds work driven by a location fix, a timer, a socket message,
a store write, or a render caused by one of those:**

- [ ] Anything repeating is gated on **application-active** state, not on
      **navigation-focus** state. Navigation focus does not change when the app leaves
      the foreground — this is the single most common cause.
- [ ] No serialisation or parsing of a payload that grows with data inside a render or a
      per-event handler. Hand components data already in the shape they need.
- [ ] No store subscription without a selector.
- [ ] No scan or sort of an unbounded array per event, and no locale-aware comparison
      where a numeric one works.
- [ ] No read-modify-write of a whole file or storage record per event. Queue and batch.
- [ ] No second writer for data a background task already records.
- [ ] For the app's most important repeating path, one real-device run with the screen
      locked, reporting mean CPU and confirming the system log holds no resource-warning
      or suspension line.

Keep a device harness in `mobile/scripts/` that replays a recorded scenario against a
real device and captures CPU, the system log, crash reports, and the app log. Without
it, this check is an opinion.

---

## 6. Authentication

- Direct authentication against the identity provider, with tokens persisted in secure
  storage.
- Refresh runs **proactively before expiry**, not reactively on a 401. A reactive refresh
  fails offline and strands the user.
- A version check at startup compares the installed build against a minimum supported
  version and prompts an upgrade below it — with a message that says what to do.

---

## 7. Testing

| Layer | What |
|---|---|
| Unit | Store actions, sync scheduler, queues, caches, pure helpers. Colocated `*.test.ts`. |
| Component | Rendering library for the platform. Native modules mocked in setup. |
| Device flows | A flow-based tool driving a real build on a simulator or device. |
| Device measurement | The harness in § 5. |

Rules:

- **Never spin up a real device for a unit test.** Mock native modules in setup.
- **The transform allowlist for untranspiled dependencies is an allowlist**, extended one
  package at a time when a test actually fails from inside it — anchored so it cannot
  pull in a neighbour.
- Cache the transform output in a repo-local directory and restore it in CI.
- The sync scheduler and the media queue are the highest-value unit tests in the package.
  Cover the backoff, the cap, the offline path, and the attempt exhaustion.

---

## 8. Builds and distribution

- Build in CI on a runner that has the platform toolchain. Keep the build job separate
  from the test job.
- A build's version and build number are derived from the pipeline, not hand-edited.
- Store metadata — descriptions, icons, screenshots — lives in the repository and is
  synced by a job, so a rejection is diagnosable from a diff.
- **Read the platform's review policies before the first submission.** Names, icons, and
  descriptions that imply an affiliation get rejected, and a rejection can cost the
  ability to distribute at all while it is appealed.

---

## 9. Compatibility with the server

Re-read [04-use-cases](04-use-cases.md) § 7 before every contract change. The mobile client is the reason
those rules exist:

- A removed field breaks an app that is still installed.
- A renamed endpoint breaks it harder.
- A changed field type corrupts a local store that already holds the old shape.

Every contract change keeps old clients working for at least one release, and the pull
request says which client version it requires and how long the overlap lasts.
