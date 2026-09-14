# 16 — Continuous integration and delivery

---

## 1. Branches

```
feature branches  →  integration branch  →  release branch
```

- **The integration branch is the default branch.** Every feature branches from it and
  merges back to it.
- **The release branch is release-only.** It moves by a deliberate merge from the
  integration branch, never by a direct commit.
- A deploy to pre-production follows a merge to the integration branch. A deploy to
  production follows a merge to the release branch.

### 1.1 Branch selection is the first act of any task

Before reading code, before planning, before editing:

1. Fetch the integration branch.
2. Measure: current branch, commits behind and ahead, working-tree status.
3. **Ask the owner what working tree to use**, with a fixed set of options, and wait.
4. Settle uncommitted changes in the same question. Every file in the status is somebody's
   work — generated files are ignored and never appear there — so each one is a second
   question, not a detail.

Never merge, rebase, force-push, reset, stash, or discard someone's work to "get current"
without being told to. Fetching and reporting is safe; changing the working tree is not.

**An option that fails is not an answer.** A fast-forward pull refuses on a diverged
branch; a new-branch command fails on a name that exists. Print the exact error and ask
again. Never pick a merge strategy yourself to make the option work.

### 1.2 Stacked branches drift

A stacked branch does not follow its base when the base takes a fix. After changing a
branch that another is stacked on, rebase the upper branches and confirm each link
explicitly.

Most hosting platforms report a stack as mergeable even when the upper branches no longer
contain their fixed bases, because each is merged against the integration branch and not
against its own base. **The drift is invisible in the place people look.**

Rebasing your own stack is fine. Force-pushing a base that someone else is stacked on is
not. Each pull request says what it was rebased onto.

**Prefer not to stack.** One pull request per unit of work, branched from the integration
branch, merged as soon as it is verified and green. Stacking is a cost paid to avoid a
merge conflict that usually does not happen.

---

## 2. Pipelines

Keep them separate by concern, and keep the trigger of each one deliberate:

| Pipeline | Trigger | Job |
|---|---|---|
| Contract check | **every pull request** | generate; fail on a violation; report the contract difference against the base |
| Unit and integration tests | integration branch, release branch, manual | all packages |
| End-to-end | manual, after a block of merges | the full browser suite |
| Deploy API | push to the branch owning the environment | build, deploy, migrate |
| Deploy clients | push, path-filtered | build and publish each client surface |
| Deploy infrastructure | push, path-filtered | plan and apply |
| Mobile build | manual or tag | platform toolchain runners |

### 2.1 Which gate actually gates

**Write down, in the repository, what gates a merge.** It is never obvious, and getting
it wrong produces confident wrong claims.

Consider each honestly:

- Does the hosting plan allow required checks on the default branch? If not, a red check
  does not block anything, and saying "CI will catch it" is false.
- Does the deploy wait for the tests, or does it start on the same push? If it does not
  wait, a test result that arrives after the deploy gates nothing.
- Do pull-request checks run against the **merge result** rather than the branch tip? If
  so, a red integration branch shows as a failure on every open pull request, and
  re-running the check cannot fix it — the fix has to land on the integration branch.

Where the honest answer is "review gates the merge, and the integration branch's own run
gates the deploy", say exactly that.

### 2.2 Concurrency

A concurrency group with cancel-on-new is right for a branch where only the newest result
matters, and wrong for the integration branch, where cancelling destroys the only record
for a merged commit.

**And know its limit:** disabling cancellation protects a run that is *executing*. It does
not protect a run that is *pending*. Most platforms keep one pending run per group, so a
newer push drops the older queued run whatever the flag says. When the runner pool is
saturated every run waits in that pending state, so every run is droppable.

Write that in a comment above the setting, because the next reader will assume otherwise.

### 2.3 Path filters

A pipeline that only runs when its paths change is efficient and is a trap: a deployable
whose path was never added to the filter silently stops deploying.

Guard it with a structural test ([12-testing-integration-and-guards](12-testing-integration-and-guards.md) § 2.1) that asserts
every deployable's path appears in the filter.

### 2.4 Self-hosted runners

Where a suite needs a machine the hosted pool does not have — a platform toolchain, a
particular browser, more I/O than a shared virtual machine gives:

- Label runners by capability, not by name.
- **Never run a suite by hand on a machine that is also a runner for that suite.** The
  two collide and the automated result is destroyed.
- Record what each runner is for, next to the workflow that uses it. Measure before
  moving a suite between pools; an I/O-bound suite can be several times slower on a
  shared volume, and the failures it produces then read as product defects when they are
  setup timeouts.

---

## 3. What CI actually checks

For each package, in this order:

1. Install.
2. **Generate** the contract artifacts, and fail on a contract violation. This runs
   before typecheck, because nothing downstream compiles until the generated client
   exists — a fresh checkout has none of it.
3. Typecheck — with the project's real typecheck command.
4. Lint.
5. Unit tests.
6. Integration tests, with the store service available.
7. Build, and confirm the artifact exists.

Then, separately: the end-to-end suite, and the infrastructure plan.

### 3.1 Reading a CI result

- **Exit zero is not the result.** Read the counts, and compare the discovered count with
  the executed count.
- **Read first-attempt failures separately** from final counts.
- **A job that reports success having discovered nothing is a failure.** A test project
  misconfigured for the runner platform reports no tests rather than an error, and every
  check it holds then gates nothing.

---

## 4. Deployment

### 4.1 API

1. **Generate the contract artifacts from the revision being deployed.** Fail the deploy
   if generation fails. Never fall back to a cached or previously built copy: a missing
   authorization table means no deploy, not an open endpoint.
2. Build every bundle, from those generated files. Confirm each expected artifact exists.
   Generate once and build from it — not once for the tests and again for the package.
3. Publish each function or container.
4. Run migrations, if the project uses them, from a single entrypoint.
5. Smoke-check: call one cheap, unauthenticated endpoint on the API host and assert a
   2xx. **A path under the application host returns the client shell with a 200 whatever
   the API is doing**, so it is never a health signal.

### 4.2 Clients

1. Build with the environment's configuration.
2. Publish fingerprinted assets with a long cache lifetime.
3. Publish the entry document with no-cache, last.
4. **Verify content, not status.** A missing file returns the application shell with a
   200.
5. Publish to **every** surface the client is served from. A project with more than one
   host serving the same application needs both updated, and a missing file on one of them
   fails silently by the previous rule.

### 4.3 Production

Production deploys are **deliberate** and owner-approved. The release is a merge from the
integration branch to the release branch, and:

- the integration branch's end-to-end run was green for the commit being released,
- the release notes are reconciled against the actual commit range,
- any migration or one-off script the release needs is listed, with who runs it and when,
- the changelog entry reaches the **release branch**, not only the integration branch, or
  production keeps serving the previous release as the newest.

---

## 5. Release notes and changelog

- One entry per production release, in the repository.
- Generated from the commit range, then edited for a human.
- It reaches the release branch as part of the release.

---

## 6. Rate limits and polling

CI APIs are rate-limited per user, and a monitoring loop burns that budget fast.

- **One watcher at a time.** Several concurrent per-run watchers polling every few seconds
  exhaust an hourly quota in minutes and then block every other API call for the rest of
  the window.
- For background monitoring, poll a list endpoint once every thirty to sixty seconds
  rather than watching each run.
- Check the remaining budget before starting a loop.
- On a rate-limit response, **stop**. Do not retry — retries queue against the same
  quota. Wait for the reset and resume more slowly.

---

## 7. Artifact storage

Build artifacts count against an account-wide quota, usually shared across repositories
and often only charged for private ones. A pipeline that uploads a large artifact per run
will hit it, and deleting artifacts may not free the quota immediately.

Upload what is needed downstream, set a short retention, and do not upload a full
dependency tree.
