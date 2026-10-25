# 27 — Tool servers and the Model Context Protocol

An AI agent reaches the application through a **tool server**: a process that turns the
API's endpoints into tools a model can call. The Model Context Protocol (MCP) is the
protocol this file uses as the reference, because it is the one the AI clients speak.

One sentence decides the whole design:

> **The tool server is a protocol translator and a token courier. It holds no privilege
> of its own.**

Every rule below follows from that sentence. The API decides who may call what, on every
call, from the user's own token. A tool server that decides anything is a second
authorization system, and two authorization systems disagree.

Reference implementation: Reference A, ADRs "Remote MCP server fronts the identity
provider with its own OAuth broker", "The hosted MCP server accepts only allowlisted
clients and asks for consent on every authorization", "The MCP keeps one active branch
per login", "Anonymous JSON staging for MCP tool calls".

---

## 1. The shape

```
model ──MCP──► tool server ──HTTPS + the USER'S token──► API ──► use case
                    │                                      │
                    │ tools built from the                 │ role gate
                    │ contract manifest ∩ allowlist        │ tenant resolution
                    │                                      │ tool-caller allowlist
                    └─ no store of its own but the session row
```

- **One tool per endpoint**, built from the generated contract manifest
  ([06-api-contract-and-codegen](06-api-contract-and-codegen.md)). A hand-written tool
  list drifts from the API on the first contract change.
- **Two transports, one core.** A local transport (standard input and output) for a
  developer's own machine, and a hosted HTTP transport for everyone else. Tool building
  is transport-agnostic: both entrypoints call one function.
- **A deliberate divergence between the transports is a named list and a test** (§ 4.3).
  An incidental divergence is a defect.
- **Tool definitions carry no per-user state.** The set is built once per process. Only
  the API client is per-session.

---

## 2. The allowlist lives in the API, not in the tool server

The dispatcher identifies "this caller is the tool server" from a **verified claim** on
the token — Reference A uses the token's audience, which is the tool server's own client
id at the identity provider. For such a caller it then enforces a **compiled-in list** of
`namespace.method` strings.

- **Compiled in, not configured.** A list in an environment variable can be changed
  without a code review. Changing a compiled list needs a deploy.
- **The tool server does not read the list.** It publishes a tool for each endpoint on
  its own copy, so the model sees nothing it cannot call; the API refuses the call
  anyway. The API's copy is the gate, the tool server's copy is a convenience.
- **Default off.** A new endpoint is off the list until someone puts it on.
- **A personal-data endpoint is never on the list.** A test fails when one is
  ([18-security-and-privacy](18-security-and-privacy.md) § 3).
- **One sentinel test pins the whole list**, so adding an endpoint to it is a visible
  diff in a security review and not a line inside a feature change.

### 2.1 Reasons to keep an endpoint off the list

These are the shapes, not one project's list. Each one has cost a reviewer a conversation:

| Shape | Why it stays off |
|---|---|
| Returns personal data | The gate exists for this. |
| Records a person's own legal act (accepting terms) | A model does not accept terms for a person. |
| Is permanent and nobody can undo it (an opt-out) | A mistaken call cannot be repaired. |
| Sets a platform limit the model itself spends against | A model must not raise its own budget. |
| Sits behind a human check (a captcha) | The tool server cannot complete it, so the tool can only fail. |
| Reaches one person's device (a push token) | The model has no device. |
| Returns an image of handwriting, or raw scans | The model must not read the content. |
| Is the readiness gate in front of an invite surface | A person must pass the gate in the application. |

Write the reason next to the exclusion, in the file that holds the list. A reader a year
later cannot reconstruct it.

---

## 3. Context is the budget

The model pays, on every turn, for every tool definition. It pays again for every byte a
tool returns. Treat both as a budget.

- **No description on a property that repeats across tools.** With hundreds of tools, one
  sentence on a shared property is that sentence hundreds of times, on every turn.
  Explain it **once**, in the server's instructions.
- **The server's instructions are built from configuration**, so each environment names
  its own URLs. A hard-coded URL in the instructions is wrong in every other environment.
- **Cap and summarise a result.** Return shape and samples, not a dump: the column
  schema, the total count, the next offset, and a bounded page. Cap on the tool side as
  well as the server side — a model that asks for 100,000 rows should get a usable answer,
  not an error.
- **Surface the number that changes the reading.** "900 uploaded" and "860 on the map"
  are different answers; the dropped count is what tells them apart.
- **A derived tool replaces a passthrough** when the passthrough returns too much, or
  when the job needs four calls in a fixed order (§ 4.2). When a derived tool takes a
  passthrough's name, publish only the derived one. The model must never see both.
- **A redacting derived tool is the only way the model sees that data.** Build the
  redaction into the derived tool and suppress the passthrough.

---

## 4. Taking a large body without making the model write it

A tool argument is written by the model, token by token, inside the call. A large body —
a layout, a document, a bulk import, a file — is slow to write, fills the conversation,
and truncates. A script on the agent's machine can build the same bytes at once, but a
tool call has no way to take its body from outside the call.

Three patterns close that gap. Pick by what the body is.

### 4.1 A large JSON body: stage it first

A script uploads the body to an anonymous staging endpoint and gets an id. The tool call
sends only the id. The API dispatcher swaps the id for the staged body, **for tool-server
callers only**, before anything else reads the body.

This is a pattern in its own right. The full design, the cost ceilings and the tests are
in **[26-staged-tool-payloads](26-staged-tool-payloads.md)**. The short form:

- The staged body is never read back by any endpoint. A leaked id lets someone *use* a
  body, never *read* one.
- The swap runs before tenant resolution and before the role gate, so no use case knows
  the pattern exists.
- The tenant never comes from inside the staged body.

### 4.2 A file: one derived tool runs the whole upload

An upload is normally four calls: create the record, ask for a signed upload form, POST
the bytes to the storage service, then finalize. Asynchronous ingestion adds a fifth: poll
until the record reaches a terminal state. Making the model do five calls in order wastes
five turns and goes wrong on the third.

**Collapse the sequence into one derived tool.** Rules, each one from a real failure:

- **Check the size before sending the bytes.** The limit comes from the API's own answer
  to the upload-form call, not from a constant the tool server copies. A copied limit
  drifts from the one the storage service enforces.
- **Delete the record you created when the upload cannot proceed.** A size refusal after
  the create leaves a record that is stuck at "awaiting upload" forever.
- **Never finalize after a failed POST.** Finalize queues ingestion of an object that is
  not there, and the record then fails for a reason that has nothing to do with the file.
- **Poll to a terminal state.** Returning "processing" leaves the model to guess, and the
  model guesses that it worked.
- **Report a poll timeout as the tool's timeout, not as the job's failure.** Name the
  record and the endpoint that reads its state, so the model can look again.
- **Bytes come in one of two ways:** base64 in the argument, for an exact small image; or
  a path on the caller's disk, which is § 4.3.

### 4.3 A file on the caller's own disk: that tool is local-transport only

A tool that reads a path with the file system cannot work on a hosted server: the hosted
process has no access to the caller's disk. **A tool that can only ever fail is worse than
an absent tool** — the model tries it, reads the error, and tries it again.

- Keep one named list of local-only tool names.
- A test asserts the hosted tool set **excludes** every name on it.
- The tool's own description says it is local-transport only.
- Say in the documentation what a hosted user does instead (upload through the web
  interface).
- Do not add to the list without the same justification. Every entry is a way the two
  surfaces differ.

---

## 5. One active tenant per login

Every tool takes an optional tenant argument, which the server sends as a header. A model
that must repeat the argument on every call eventually omits it, and an omitted argument
is silently the caller's home tenant. In a conversation about tenant B, one missed
argument on a write puts the row in tenant A.

**Keep one active tenant per login**, on the session row the hosted server already reads.

- **Precedence:** the per-call argument, then the stored value, then no header. Read the
  store only when the call named no tenant.
- **The stored value grants nothing.** The API judges the header on every request. A
  membership revoked after the switch refuses the next call.
- **Set it through an endpoint already on the allowlist**, and store it only when the
  answer confirms the switch. Do not add an endpoint for it.
- **On a refusal that names the tenant, clear the stored value, report it, and do not
  retry on the home tenant.** Match the **message**, not the status: an ordinary
  permission refusal must not throw a switched caller back to their own tenant.
- **Every result that used the header names the tenant it ran against**, in a second
  content block. The value is shared by every conversation on that login, so a result
  that does not name its tenant is readable as the wrong tenant's.
- **One read per call.** Hold the session row for one round and share it between the
  token source and the tenant store.
- **The tools that set and read the active tenant are exempt from the tenant argument.**
  That exemption is one named list, and a test asserts its contents.
- **The API refuses a tenant that does not exist**, and does not fall back. A shape check
  on the identifier is not an existence check.

---

## 6. The authorization lifecycle

This section is the hosted transport. The local transport is § 6.5.

### 6.1 Why the server runs its own authorization surface

The protocol expects the tool server to be an OAuth 2.1 protected resource: the client
reads the protected-resource metadata, finds the authorization server, performs
**dynamic client registration**, then an authorization-code flow with PKCE.

Most managed identity providers do **not** implement dynamic client registration, and
their clients are provisioned by the infrastructure. So the tool server cannot point the
AI clients at the identity provider and step aside.

**The broker facade.** The tool server runs its own minimal authorization-server surface
(metadata, registration, authorize, token, revoke) and delegates the actual
authentication to the identity provider's hosted sign-in page. It issues its own opaque
tokens, and maps each one to the stored identity-provider tokens.

**Read this as the narrow exception it is.** The standing rule is the managed identity
service for all authentication, enforced at the edge, with nothing hand-rolled
([18-security-and-privacy](18-security-and-privacy.md) § 1). This is the one place that
rule bends; it bends only because of the missing dynamic client registration; and it bends
as little as it can — the SDK owns every primitive, the user still authenticates on the
provider's own hosted page, the tool server never sees a password, and the API still
decides every authorization from the provider's own token. **Do not read this file as a
licence to write an authorization server anywhere else.**

- **Build the facade on the protocol SDK's own router and provider interface.** PKCE
  verification, redirect-URI validation, client authentication, registration and the
  basic rate limits are then SDK code. What the repository adds is the brokering.
- **Hand-rolled OAuth primitives and hand-rolled cryptography are out of bounds.**
- **Keep one upstream client id.** The API identifies the tool caller from the token's
  audience. A second upstream client means a second audience, and the API's gate must
  become a set membership check. That is the single change this design exists to avoid.
- **Run the transport stateless.** Continuity lives in the access token and the session
  store, not in transport state, so consecutive requests may land in different execution
  environments. A cache in front of the store goes stale across them.

### 6.2 The flow, step by step

| # | Step | What the server does |
|---|---|---|
| 1 | Discovery | Serves protected-resource metadata and authorization-server metadata from the exact public HTTPS base URL. |
| 2 | Registration | Refuses a client whose every `redirect_uri` is not on the allowlist, with `invalid_client_metadata` and HTTP 400. Refuses when the list is empty. |
| 3 | Client read | Drops stored redirect URIs that are off the list, so a client registered before the list cannot use one. |
| 4 | Authorize | Checks the allowlist again, and writes the 400 **itself**. It never redirects to an off-list URI. Refuses a resource indicator that is not this server. Parks the request with a short TTL, and starts a **separate** PKCE exchange against the identity provider. |
| 5 | Upstream callback | Exchanges the code. Checks that the returned identity token's audience is the expected client. Saves a **pending consent** holding the upstream tokens and the **hash** of a CSRF token, with a short TTL. Renders the consent page from the server's own origin. It does not redirect to the AI client here. |
| 6 | Consent | § 6.3. Approve opens the session and mints a short-lived code. Deny deletes the pending consent and redirects with `error=access_denied`. |
| 7 | Token | Issues the server's **own** access and refresh tokens, stored as SHA-256 hashes against the session row. The row holds the upstream tokens. |
| 8 | Each call | Bearer check against the expected resource, one read of the session row, the upstream identity token forwarded to the API. |
| 9 | Refresh | Refreshes upstream transparently. A refresh record that names no resource is refused: re-authenticate. |
| 10 | Revoke | Deletes by token hash. |

### 6.3 The consent page

A proxy authorization server that uses one static upstream client, accepts open
registration, and sits behind an upstream single-sign-on cookie has the **confused deputy**
problem: an attacker registers a client with their own redirect URI, sends the user an
authorize link, the user's browser completes the flow with no prompt, and the code goes to
the attacker. The protocol's security guidance says such a server **must** show its own
consent before it releases a code.

- **The page shows the display name from the allowlist, never the name the client
  registered.** A client chooses its own name; the allowlist does not.
- It also shows the **redirect host**, and the **account identity and roles** that are
  about to be handed over. The account name is the signal a phished user needs.
- A loopback redirect matches any program on the user's own computer. The page says so.
- **Approve is a POST** carrying the CSRF token. Check the token **before** the single-use
  take, so a forged post does not consume the consent.
- **No session exists before Approve.** A denial leaves nothing behind.
- **Every authorization asks again.** Remember no approval.
- Headers: `X-Frame-Options: DENY`, a Content-Security-Policy with `frame-ancestors
  'none'`, and `Cache-Control: no-store`. Do **not** set `form-action`: browsers apply it
  to the redirect that follows the form post.
- Escape every printed value.
- A GET on the consent path answers 405, never a redirect. Every failure is an HTML page
  on the server's own origin.

### 6.4 The rules that make the rest of it safe

- **The redirect-URI allowlist and the consent page are both needed.** The allowlist stops
  a code reaching an attacker's server. The consent page stops a silent flow to a local
  program and a phished approval. Neither covers the other's case.
- **The allowlist is a code change.** A new AI client cannot connect until the change
  deploys, and a vendor that moves its callback breaks until the list follows. No
  automated test can prove a vendor still uses its URI: that check is manual, and the
  pull request records it.
- **Bind every token to this server's own resource.** A token issued for another resource
  is refused at the bearer check.
- **Store the server's own tokens as hashes.** A dump of the table then yields nothing
  usable.
- **Encrypt the upstream tokens at rest with a customer-managed key**, put a TTL on the
  session row, and never log a token. Log the message of a failed exchange, not its body.
- **Parse and size-limit the request body before validation**, so an oversized body is
  refused with 413 and never reaches the protocol layer.
- **Trust exactly one proxy hop** when keying a rate limit on the client address. Trusting
  every hop lets any caller spoof the header and walk past the limit. Behind a content
  delivery network, resolve the viewer address from the signed header the distribution
  adds, before the rate limits run.
- **Lifetimes, as values to copy and then tune:** parked authorization 10 minutes, pending
  consent 10 minutes, authorization code 5 minutes, access token 1 hour, refresh token the
  same as the upstream refresh token.
- **No standing privilege.** The server's cloud permissions are its own session table and
  that table's key. It has no database access to the application's data and no
  administrative permission at the identity provider, so a compromise of it grants no more
  than the allowlist already grants each authenticated user.
- **Test controls stay out of the deployed entrypoint.** Where the local test stack needs
  routes that write the store directly, mount them only for the in-memory store, and never
  from the deployed handler.

### 6.5 The local transport

- The same core, the same tool building, the same upstream client id and therefore the
  same audience at the API.
- A loopback redirect with PKCE, and a token file with tight permissions on the user's own
  machine.
- The active tenant lives in process memory rather than on a session row.
- Keep it working. It is the development path, and it is the only transport that can read
  the caller's disk (§ 4.3).

---

## 7. What a tool server must never do

| Never | Why |
|---|---|
| Decide what the caller may call | The API decides, on every call. A second decision point disagrees with the first. |
| Hold a service credential that reaches the data | Then a compromise of the tool server is a compromise of every tenant. |
| Read a staged payload back | A read path turns a leaked id into a data leak. |
| Publish a tool that cannot work on its transport | The model tries it, and tries it again. |
| Put a secret in a tool result or an error | The result is conversation text. |
| Swap a staged body for a browser or mobile caller | That opens a second body path nothing needs. |
| Remember an approval | Every authorization asks again. |

---

## 8. Tests

**Tool building** — in-memory, no network ([11-testing-unit](11-testing-unit.md)):
- the published set is the manifest intersected with the allowlist;
- a derived tool suppresses the passthrough of the same name, and the set holds one of them;
- the local-only names are present on the local transport and **absent** on the hosted one;
- a tool with no body gets no staged-payload property;
- the instructions name the configured base URL, the body shape and the limits.

**The allowlist** ([12-testing-integration-and-guards](12-testing-integration-and-guards.md)):
- a sentinel pins the whole list;
- no endpoint marked as returning personal data is on it;
- a call to a method off the list is refused by the API, not by the tool server.

**The upload tool**:
- refuses a file over the limit the API returned, **before** it reads the file, and deletes
  the record it created;
- does not finalize after a failed POST;
- polls to a terminal state, and reports its own timeout as a timeout.

**The active tenant**:
- the per-call argument wins over the stored value, and the stored value over no header;
- the store is read only when the call named no tenant;
- the two named refusal messages clear the stored value; an ordinary permission refusal
  does not;
- every result that used the header names the tenant and its source;
- the exempt tool list has exactly the expected names.

**The authorization surface**:
- registration refuses an off-list redirect URI, and refuses an empty list;
- authorize refuses an off-list URI with its own 400 and no redirect;
- the upstream callback renders the consent page and redirects to no client;
- approve needs the CSRF token, and a wrong token does not consume the consent;
- an unknown, expired or used consent id answers 400 as an HTML page;
- a GET on the consent path answers 405;
- deny leaves no session;
- a token issued for another resource is refused at the bearer check;
- an oversized body answers 413 before validation;
- a refresh record naming no resource is refused;
- revoke deletes by hash;
- the forwarded token's audience is the expected client — assert this, because the API's
  whole gate depends on it.

**End to end** ([13-testing-e2e](13-testing-e2e.md)): a real client completes
registration, authorize, consent and a tool call against a running server and a running
API; a method off the allowlist is refused; a staged body cannot move a write into another
tenant.

---

## 9. Swap points

- **Protocol:** MCP or any other tool protocol. The allowlist, the staged-payload swap and
  the tenant header live in the API, so the tool layer only publishes and translates.
- **Transport:** standard input and output, HTTP, or both. The core is shared.
- **Identity provider:** any that supports an authorization-code flow with PKCE. The
  broker exists because of the missing dynamic client registration; a provider that
  supports it removes the facade and keeps everything else.
- **Session store:** any key/value store with TTL, or a table with an expiry column.
- **Compute:** request/response serverless is enough while the tool surface has no
  server-initiated notifications. A persistent stream needs a long-lived host.

**Not swap points:** the allowlist in the API, the user's own token on every call, no
standing privilege, consent before every code, the redirect-URI allowlist, tokens stored
as hashes, and the transport divergence being a named list with a test.

---

## 10. Adopting it

1. Generate the tool set from the contract manifest, and publish nothing outside it.
2. Add the allowlist to the API dispatcher, keyed on a verified token claim. Add the
   sentinel test and the personal-data test.
3. Stand up the local transport first. It needs no hosting and proves the tool surface.
4. Add the derived tools that collapse multi-call sequences and cap large results.
5. Add staged payloads ([26-staged-tool-payloads](26-staged-tool-payloads.md)).
6. Add the active-tenant tools and the precedence rule.
7. Build the hosted transport on the SDK's authorization router: metadata, registration,
   authorize, token, revoke.
8. Add the redirect-URI allowlist at registration, client read and authorize.
9. Add the consent page, its CSRF token, its headers and its single-use take.
10. Add the session table with a TTL, a customer-managed key, and no permission beyond it.
11. Write the tests in § 8, and see each one fail once for the right reason.
12. Do the manual check against each real AI client, and record it in the pull request.
