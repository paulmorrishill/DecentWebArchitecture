# 15 — Infrastructure and environments

Everything deployed is declared in code, reviewed as a plan, and applied by a pipeline.
Nothing is created by hand in a console.

---

## 1. Layout

```
infra/
  modules/
    api/                  the API host, its role, its tables, its queues
    static-site/          bucket + CDN + certificate for a client
    identity/             user pool, groups, clients
    domain/              zone, records, certificates
    push/                 socket API, fan-out, connection table
    <feature>/            one module per cohesive piece of infrastructure
  environments/
    staging/              module instances + variable values
    production/
  scripts/                one-off operational scripts
  README.md               module map and conventions
```

- A module is a cohesive unit with a clear boundary, not a folder per resource.
- An environment is a thin composition of modules with its own variable values. It
  contains no resources of its own beyond the wiring.
- Each environment has its **own isolated state** and its own credentials.

---

## 2. Environments

Three, at most:

| Environment | Fed by | Purpose |
|---|---|---|
| Local | a developer machine | the simulators in [14-local-dev-and-simulators](14-local-dev-and-simulators.md) |
| Pre-production | the integration branch | the real cloud, real deploys, disposable data |
| Production | the release branch | real users |

Rules:

- Pre-production and production are **separate accounts or projects**, not separate
  prefixes in one. Separation of credentials is what stops a mistake crossing over.
- Because each environment has its own account, resource names generally need no
  environment prefix — and a prefix that varies by environment makes every reference and
  every script conditional. **Exception:** globally-unique namespaces (object-storage
  bucket names, public hostnames) must be prefixed, and only at creation time. Existing
  resources keep their names.
- Configuration differences between environments are **values**, never conditional
  resource blocks. A resource that exists in one environment and not another is two
  different systems.

---

## 3. Naming

Write the rules down in the infrastructure readme, once, and let a guard test enforce
what it can:

- One naming convention per resource type, derived from a small set of inputs (service,
  component, environment).
- **Check the length limits.** Different services enforce different maximum name
  lengths, and a name that is one character over fails at apply time — after other
  resources have already been created.
- Derive names from a shared local value rather than repeating a string. A guard test
  that quotes a literal name will then break the day someone does this correctly, so
  guard on the **derived value**, not on the literal.

---

## 4. Reviewing a change

**The plan is the review.** Nothing merges without one being read.

- [ ] Read **every** line marked as forcing replacement. None should be a surprise.
- [ ] A resource you intended to change in place shows as an in-place update. If it shows
      as a replacement, stop and find out why.
- [ ] **No resource carrying state is in the destroy list** — a table, a bucket, a queue,
      a database, a user pool. Ever.
- [ ] If the plan cannot be run locally, the pull request says so explicitly, so the
      reviewer runs it.

### 4.1 Replacements that destroy data

For most infrastructure tools, several ordinary-looking edits mean destroy-and-recreate,
which for a data store is total loss:

| Edit | Consequence |
|---|---|
| Renaming the **resource label** | Destroys and recreates. Use the tool's move/rename mechanism, or keep the label. |
| Changing a key attribute | Forces replacement. Plan a parallel table and a backfill. |
| Changing a table-level billing or capacity mode | May force replacement. Check. |
| Adding an index | Usually online and safe. |
| Removing an index any code path queries | Runtime failure. Remove the callers, ship, then remove the index. |
| Adding a non-key attribute | Safe on a schemaless store. |
| Renaming an attribute on existing rows | Breaking. Dual-write for a release, or migrate first. |

### 4.2 Quotas

Cloud providers enforce quotas that are invisible until an apply fails — the number of
managed policies attachable to a role, the number of rules on a distribution, the number
of resources in a stack.

- Know the ones your design pushes against and record them in the infrastructure readme.
- Prefer the form that does not consume the quota (an inline policy rather than a managed
  one, a merged rule rather than a new one).
- Avoid a merge-and-delete in a single apply. The tool may parallelise the detach and the
  attach, and a quota check at attach time can race with the detach. Run two applies, or
  use the form that sidesteps the quota.

### 4.3 Cycles

A resource that references a resource that references it back will not plan. The common
shape is a compute resource and its own log destination. Break the cycle by naming the
dependent resource from a derived string rather than from the other resource's
attribute.

---

## 5. Applying

- Applies happen in a pipeline, triggered by a merge to the branch that owns the
  environment.
- The pipeline's credentials come from a short-lived federated identity, not from a
  long-lived stored key.
- **Manual applies are for emergencies and are announced.** An out-of-band apply leaves
  the state ahead of the branch, and the next pipeline run reverts it.

---

## 6. Secrets

- Never in the repository. Not in a test fixture, not in a comment, not in a
  configuration default.
- Injected at deploy time from the pipeline's secret store, or read at runtime from a
  parameter store.
- A hardcoded secret discovered in the codebase is rotated, and rotating it is a
  coordinated change: move it to a parameter, deploy, verify, then remove the literal.
- Document how to rotate each one. An undocumented rotation is an outage waiting for the
  day it is needed.

---

## 7. Cost

- An alert on unexpected spend, per environment, wired in the same infrastructure.
- Pin retention on logs. An unbounded log group is the most common surprise line on a
  bill.
- Know which resources bill per request and which bill per hour. A design that adds a
  per-item call inside a loop changes the shape of the bill, not just the latency.

---

## 8. Static content and content delivery

- Each client surface is a bucket plus a distribution plus a certificate, produced by one
  module.
- **A missing file in a single-page application returns the application shell with a 200.**
  So "the file deployed" cannot be checked by fetching it and looking at the status.
  Check the content.
- Cache headers: fingerprinted assets get a long maximum age; the entry document gets
  no-cache, or the client never picks up a new bundle.
- Where a distribution has a limited number of routing behaviours, that limit is a design
  constraint on how many origins one surface can have. Know the number before designing
  around it.
- A rewrite rule that changes a path does not change which origin serves it. Check both.

---

## 9. Schema and data in infrastructure

Where table definitions live in infrastructure ([05-repositories-and-persistence](05-repositories-and-persistence.md) § 5),
the **same definitions** are imported by the local host so the local shape cannot drift.
Put them in one module both sides read, rather than writing them twice.

Where schema lives in migrations, the deploy runs them, and the pipeline records which
migrations ran.
