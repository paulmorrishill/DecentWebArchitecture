# 18 — Security and privacy

---

## 1. Authentication

- An external identity provider issues tokens. The application never stores a password.
- The gateway or the entrypoint **verifies** the token before any application code runs.
  Claims reaching the request context are verified claims.
- The client parses the token only for display and routing decisions. **Never trust a
  client-side claim for an authorization decision.** Every authoritative check is
  server-side.
- Tokens travel in headers. **Never in a URL, a query string, or a log line.**
- Storage: browser local storage is acceptable for a token the server refreshes on each
  response; a mobile app uses the platform's secure storage. Pick one per client and
  document the refresh mechanism.
- An unauthenticated response clears the credential and routes to sign-in.

---

## 2. Authorization

Three layers, each doing one job:

**1. Role gate.** The generated route table carries the endpoint's required roles. The
dispatcher checks them before the use case runs. A wildcard administrative role satisfies
every requirement, defined in one place.

**2. Ownership and tenancy.** Inside the use case, or enforced by the scoped repository
base class. A caller with the right role can still be asking about a record that is not
theirs.

**3. Field-level filtering.** What the response includes can depend on the caller's role.
Do **not** share one response type across roles — a separate endpoint or a separate
response type per role is clearer and cannot leak by omission of a filter.

Rules:

- **A public endpoint declares the anonymous role explicitly.** An empty role list is
  indistinguishable from an omission.
- **Tightening roles requires regenerating the route table**, or the running system keeps
  the old, looser rule. Fail-open, and silent.
- **Widening roles requires re-auditing the response** for data the newly-permitted role
  must not see. Fields that were safe before may have grown new neighbours.
- **Every role-gated surface gets a test both ways** — usable for a permitted role, absent
  or refused for an unpermitted one — at whichever layer the gate lives.

---

## 3. Personal data

The rule that makes this enforceable rather than aspirational: **every endpoint declares
whether it can return personal data about a real person, and the declaration is required.**

- The flag is part of the endpoint metadata. Omitting it is a compile error. A
  non-literal value fails extraction.
- The extractor produces a table the dispatcher reads at runtime to gate those responses.
- Because it is required, a new endpoint cannot default into the permissive answer.

Beyond the gate:

- **Do not return personal data a screen does not display.** A response shaped for an
  administrator, reused by a lower-privileged screen, leaks in the network tab whether or
  not anything renders it.
- **Reference personal data by identifier** where a screen only needs to know that a
  record exists.
- **Redact in support exports and diagnostics** — precise location, contact details,
  free-text notes.
- **Never place personal data in a URL**, a query string, a log line, or an error
  report's tags.
- A public, unauthenticated asset is world-readable to anyone who guesses its path. Person-
  level data never goes there. See [08-multi-tenancy](08-multi-tenancy.md) § 5.

---

## 4. Input handling

- **Validate at the use case.** Repositories assume valid input.
- **Parameterise every query.** No caller input concatenated into a query string, in any
  store.
- **Rate-limit public unauthenticated endpoints** — sign-up, contact forms, public
  submissions — keyed by the caller's address, with per-feature limits.
- **Verify webhook signatures before any side effect**, not after.
- **Treat inbound mail and third-party payloads as untrusted input**, including their
  file names and content types.
- **Uploads** go to object storage through a time-limited URL issued by an authorized
  endpoint, with the content type and a size limit pinned in the signature.

---

## 5. Least privilege

- One identity per deployable, with only the permissions that deployable uses.
- Permissions are declared in infrastructure alongside the resource they refer to.
- A new permission is a reviewed line in a pull request, never a console change.
- Where a quota limits attached policies, prefer the inline form rather than removing the
  separation ([15-infrastructure-and-environments](15-infrastructure-and-environments.md) § 4.2).
- The deploy pipeline authenticates through a short-lived federated identity, not a
  long-lived stored key.

---

## 6. Secrets

- Never committed. Not in a fixture, not in a comment, not as a default.
- Injected at deploy time or read from a parameter store at runtime.
- Documented rotation procedure per secret.
- A secret found in the codebase is rotated in a coordinated change: move to a parameter,
  deploy, verify, then remove the literal — never rotate first and break the running
  system.

### 6.1 Asking a human for a credential

**Never ask for a secret in a chat, a transcript, or a log.**

Use a mechanism where the human types the value into a local prompt and it lands only on
disk at a path the agent names. Then:

- Parse it defensively, accepting the shapes a human actually pastes.
- **Redirect every parse error to nowhere.** A failed assignment in most shells prints
  the whole offending line, which is the secret.
- Verify the identity it grants and print only the account identifier, never the value.
- Stop if the account does not match the intended target.
- Delete the file when the work is done.

---

## 7. Operating on production

- **Production is data-blind.** An operational script reads the data; the operator sees
  counts. Filter output to summary lines and never print item identifiers, names, or
  rows. Pre-production may show item lines.
- Unset any local endpoint overrides before running a script against a real environment,
  or it silently operates on the local simulator and reports success.
- Verify the target account before the first write, and stop on a mismatch.
- Every script is idempotent and re-runnable. A single-shot script that fails halfway
  leaves a mixed state with no recovery path.
- A dry-run mode that reports what it would do, by count, is the default.

---

## 8. Dependencies

- Pin versions. A floating major is an unreviewed change in every build.
- Adding a dependency to a client is an architectural decision — it ships to every user.
- Prefer the platform over a package for anything small.
- Keep a scheduled job that reports known vulnerabilities, and treat its output as work,
  not as noise.

---

## 9. Things that quietly become security surfaces

| Surface | Why |
|---|---|
| A generated authorization or privacy table | Stale means fail-open, silently |
| A tool-server endpoint allowlist | Decides which endpoints a machine client can reach |
| A cross-origin configuration | Too permissive means any site can call the API with the user's credential |
| A public bucket path | Separated by nothing but the path |
| An anonymous endpoint's tenant resolver | Missing means either a failure or the wrong tenant |
| A development-only shim | One inverted condition from running in production |
| A signed upload URL | Its content type and size limit are the only constraint on what lands |

**No development-only shim changes shipped behaviour.** A build-flag or hostname gate that
bypasses authentication, sync, or a network call is one inverted condition away from
doing it for everyone. Existing ones are test affordances; a new one must not touch an
authentication, sync, or network path.

---

## 10. Features that hand the user to a third party

**A feature whose implementation is an outbound link to a console you do not control gets
one manual pass against the real destination before it counts as done.**

The automated suite can assert the anchor renders and the address looks plausible. It
cannot follow the link or check the far side, so a link pointing at a page that does not
exist — or carrying the wrong parameters — passes every test and fails every user.

- [ ] Someone follows the link once, against the real destination, and confirms the page
      exists and does what the feature claims.
- [ ] If the address is hand-built, a unit test pins the parameters the far side requires.
- [ ] The pull request says who took the manual pass and what they saw.
