# 26 — Staged payloads for large agent tool calls

An AI agent that calls the API through a tool server (an MCP server, or any tool layer
that turns endpoints into model tools) must write every tool argument token by token,
inside the call. A large body — a map layout, a long document, a bulk import — is slow
to write, fills the conversation, and is easy to truncate. A script on the agent's
machine can build the same JSON at once, but a tool call has no way to take its body
from outside the call.

This file is the pattern that closes that gap: **a script uploads the body to an
anonymous staging endpoint and gets an id back; the tool call sends only the id; the
API dispatcher swaps the id for the staged body, for tool-server callers only.**

Reference implementation: Reference A, ADR "Anonymous JSON staging for MCP tool calls".

This file is one pattern of the tool server. The tool server itself — the allowlist, the
context budget, file uploads, the per-login tenant and the whole authorization lifecycle —
is [27-tool-servers-and-mcp](27-tool-servers-and-mcp.md).

---

## 1. The flow

```
agent ──writes──► script ──POST /tool-payloads/stage {payload}──► API (anonymous)
                                │                                   │ size check
                                │◄──── { stagedPayloadId, expiresAt }┤ store row, TTL
                                ▼
agent ──tool call { stagedPayloadId }──► tool server ──POST /orders/import { stagedPayloadId }
                                                          + the tool server's token
                                                                    │
                                              dispatcher: caller is tool server?
                                                yes → body := staged JSON
                                                no  → body unchanged
                                                                    │
                                              tenant resolution, role gate, use case
```

The model writes a short script and one id. The script never needs a token, because the
tool server holds the login, not the script.

---

## 2. The staging endpoint

An ordinary use case behind an ordinary RPC path
([04-use-cases](04-use-cases.md), [06-api-contract-and-codegen](06-api-contract-and-codegen.md)):

```ts
@ApiEndpoint({ namespace: 'toolPayloads', method: 'stage', roles: ['Anonymous'], personalData: false })
export class StageToolPayloadUseCase {
  constructor(
    private readonly repository: StagedToolPayloadRepository,
    private readonly clock: Clock,
    private readonly newId: () => string = randomUUID,
  ) {}

  async execute(request: StageToolPayloadRequest): Promise<StageToolPayloadResponse> {
    const payload = request?.payload;
    if (typeof payload !== 'object' || payload === null || Array.isArray(payload)) {
      throw new ValidationError('`payload` must be a JSON object.');
    }
    const payloadJson = JSON.stringify(payload);
    if (Buffer.byteLength(payloadJson, 'utf8') > MAX_STAGED_PAYLOAD_BYTES) {
      throw new ValidationError(`\`payload\` is larger than ${MAX_STAGED_PAYLOAD_BYTES} bytes.`);
    }
    const stagedPayloadId = this.newId();
    const now = this.clock.now();
    const expiresAt = new Date(now.getTime() + STAGED_PAYLOAD_LIFETIME_MS).toISOString();
    await this.repository.save({ stagedPayloadId, payloadJson, createdAt: now.toISOString(), expiresAt });
    return { stagedPayloadId, expiresAt };
  }
}
```

Rules:

- **Request:** `{ "payload": <the tool arguments object> }`. The payload must be a JSON
  object, because a tool body is an object.
- **Response:** `{ stagedPayloadId, expiresAt }` and nothing else. **No endpoint ever
  reads a staged payload back.** Without a read path, a leaked id lets someone *use* a
  body, never *read* one, and the endpoint can be classified as returning no personal
  data.
- **The id is made on the server**, from a random UUID (122 random bits). Never accept a
  caller-chosen id: that turns the table into a namespace someone can squat or overwrite.
- **Size is counted in UTF-8 bytes, not characters.** The cap sits below the store's item
  limit with room for the other attributes (Reference A: 400,000 bytes against a 400 KB
  item).
- **Write with a not-exists condition on the key**, so a collision fails instead of
  replacing a row.
- **Do not log the payload.** It is the caller's tool body and can hold anything.

### 2.1 The table

- **Global, not tenant-scoped.** The upload is anonymous, so there is no tenant to scope
  by, and the key is a server-made UUID. Write this down as the global-data exception
  ([08-multi-tenancy](08-multi-tenancy.md)).
- **Short lifetime.** Reference A: ten minutes. Long enough for a retry, short enough
  that the table holds almost nothing.
- **The store's TTL is the only delete**, and TTL is lazy (DynamoDB can delete up to two
  days late). **The reader checks `expiresAt` on every read.** Never treat "the row
  exists" as "the row is valid".
- **No point-in-time recovery and no backup export** for this table. Every row is gone
  within minutes, so a restore has nothing to give back, and a backup keeps a copy of
  every payload for weeks. Pin this with a guard test
  ([12-testing-integration-and-guards](12-testing-integration-and-guards.md)).

---

## 3. The swap, in the dispatcher

The swap is one function, called right after the entrypoint parses the body and **before
anything else reads the body** — before tenant resolvers, before the role gate, before
the use case. Every later step then sees the staged JSON as if the tool server had sent
it, so no use case knows the pattern exists.

```ts
export const STAGED_PAYLOAD_ID_FIELD = 'stagedPayloadId';
const UUID_PATTERN = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/;
const NOT_FOUND = 'The staged payload does not exist or has expired. Stage it again.';

export async function swapStagedPayload(
  body: Record<string, unknown>,
  isToolServerCaller: boolean,
  repository: StagedToolPayloadRepository,
  nowIso: string,
): Promise<Record<string, unknown>> {
  if (!isToolServerCaller || !(STAGED_PAYLOAD_ID_FIELD in body)) return body;
  if (Object.keys(body).length > 1) {
    throw new ValidationError('A body with `stagedPayloadId` must have no other field.');
  }
  const id = body[STAGED_PAYLOAD_ID_FIELD];
  if (typeof id !== 'string' || id.length === 0) {
    throw new ValidationError('`stagedPayloadId` must be a non-empty string.');
  }
  if (!UUID_PATTERN.test(id)) throw new NotFoundError(NOT_FOUND);

  const staged = await repository.findById(id);
  if (!staged || Date.parse(staged.expiresAt) <= Date.parse(nowIso)) {
    throw new NotFoundError(NOT_FOUND);
  }
  return JSON.parse(staged.payloadJson) as Record<string, unknown>;
}
```

```ts
// entrypoint, straight after parsing
const payload = await swapStagedPayload(
  parseJsonObject(event.body),
  Boolean(TOOL_SERVER_AUDIENCE) && caller.audience === TOOL_SERVER_AUDIENCE,
  stagedPayloadRepository,
  clock.nowIso(),
);
```

Rules:

- **Only the tool server's callers get the swap.** Decide it from a verified token claim
  (Reference A: the token's audience is the tool server's client id). For a web or mobile
  token the body is unchanged, so the id reaches the use case as an unknown field and the
  new data path never reaches the apps. Guard the empty-configuration case: an unset
  audience must mean "nobody", never "everybody".
- **A body with the id and any other field is a 400.** Mixing is ambiguous — which wins? —
  so refuse it. The one exception is a tenant selector the tool server adds outside the
  body (§ 3.1).
- **Unknown, malformed or expired id → 404** with one message that tells the agent what to
  do: stage it again. Do not tell the caller *which* of the three it was.
- **Check the id's shape before the read.** A string that cannot be a server-made id names
  no row. Without the check, a 3 KB id reaches the store, the store refuses the key with a
  validation exception, and that reports as a server fault.
- **The id is reusable until it expires and is not bound to a caller.** A failed tool call
  can be retried with the same id. The cost is § 5.
- **The swapped body runs under the second caller's roles.** The role gate runs after the
  swap, against the token that sent the id, not anything about who staged it.

### 3.1 The tenant stays outside the staged body

The tenant comes from the token, or from the tool server's own tenant argument, never from
inside the staged JSON. A staged body that names another tenant cannot move a write there.
Prove that with an end-to-end test (§ 7) — it is the cross-tenant test
[08-multi-tenancy](08-multi-tenancy.md) requires, applied to the new path.

---

## 4. The tool schema

The tool server publishes one tool per endpoint from the generated contract manifest
([06-api-contract-and-codegen](06-api-contract-and-codegen.md)). For every tool that takes
a body, it adds an optional `stagedPayloadId` property:

```ts
export function withStagedPayloadIdProperty(schema: JsonSchema): JsonSchema {
  if (schema.properties && STAGED_PAYLOAD_ID_PROPERTY in schema.properties) {
    throw new Error(`Endpoint schema already declares "${STAGED_PAYLOAD_ID_PROPERTY}".`);
  }
  const { required, ...rest } = schema;
  return {
    ...rest,
    properties: { ...(schema.properties ?? {}), [STAGED_PAYLOAD_ID_PROPERTY]: { type: 'string' } },
    ...(required && required.length > 0
      ? { anyOf: [{ required }, { required: [STAGED_PAYLOAD_ID_PROPERTY] }] }
      : {}),
  };
}
```

- **The endpoint's `required` list moves into an `anyOf`.** Without that, a call with only
  the id fails the client's schema check before it is sent.
- **Refuse a schema that already declares the property.** The staged argument would shadow
  a real field.
- **No description on the property.** With hundreds of tools, a description repeats in
  every tool definition and costs context on every turn. Explain it **once**, in the tool
  server's instructions, built from the configured API base URL so each environment names
  its own staging URL:

  > Large tool arguments: a tool call whose arguments are too large to write in the call
  > can send `{ "stagedPayloadId": "<id>" }` in place of every other argument. To get the
  > id, a script sends `POST <api>/tool-payloads/stage` with the JSON body
  > `{ "payload": <the tool arguments object> }` and no Authorization header. The answer is
  > `{ "stagedPayloadId": "<id>", "expiresAt": "<ISO time>" }`. The payload is at most
  > 400,000 bytes of JSON. The id can be used again for 10 minutes, then answers 404. One
  > IP address can stage 20 payloads in 5 minutes.

- **Tools with no body do not get the property.**
- **A hand-written tool that replaces a generated one does not get it automatically.**
  Add it by hand or accept the gap, and say which.

---

## 5. Cost ceilings

An anonymous write is paid for by the platform. Put a ceiling on each axis, and write the
worst-case monthly number in the decision record
([19-performance-and-scale](19-performance-and-scale.md) § 9).

| Axis | Control | Reference A |
|---|---|---|
| Bytes per call | Size check in the use case | 400,000 bytes |
| Calls per IP | Edge rate rule scoped to the stage path, ordered before the general anonymous rule so it counts first; answers 429 | 20 per 5 minutes |
| All IPs together | Table-level write throughput cap | 400 write units/s |

The per-IP rule does not stop many addresses together. The table cap does: above it the
store refuses the write, so the bill cannot pass `cap × seconds in a month × unit price`.
The price of the cap is that staging fails during such an attack. Normal tool calls are
unaffected, because they do not need staging.

Reference A's numbers: normal use (1,000 payloads a month) is about £0.45; one IP at its
limit for a month is about £76; the table cap bounds all IPs together at about £577.

---

## 6. Options rejected

| Option | Why not |
|---|---|
| Swap for every caller | Opens a second body path for web and mobile clients that nothing needs, and widens the attack surface of an anonymous write. |
| Swap in the tool server (read back, then send) | Needs a read-back endpoint, which turns a leaked id into a data leak. Needs one change per tool-server transport (local and hosted) instead of one in the API. |
| Authenticated staging | The script would need a token, and the token lives in the tool server. Handing it to a script is the thing the pattern exists to avoid. |
| Object storage with a presigned upload URL | Works, but needs a tool call to get the URL first, a second store to clean up, and the same swap anyway. Prefer it only when payloads must pass the item-size limit. |

---

## 7. Tests

Use-case and service tests use in-memory fakes ([11-testing-unit](11-testing-unit.md)):

**Stage use case**
- needs no role;
- stores the payload as JSON text and answers the id and the expiry;
- makes a random UUID by default;
- accepts a payload of exactly the size limit;
- refuses a payload one byte over, and stores nothing;
- counts the size in UTF-8 bytes, not characters (a multi-byte character at the limit);
- refuses a request with no body, and a payload that is an array, a string or null.

**Swap**
- replaces a tool-server body that holds only the id with the staged JSON;
- lets the same id be used again before it expires;
- returns the body unchanged for a caller that is not the tool server;
- returns a tool-server body with no id unchanged;
- refuses an id sent with any other field;
- answers not-found for an unknown id, and for a malformed id without reading the store;
- answers not-found **at** the expiry time, while the row is still in the table.

**Tool schema**
- adds the property and moves `required` into an `anyOf`;
- adds no `anyOf` when the endpoint requires nothing;
- refuses a schema that already declares the property;
- keeps the tenant argument addable on top;
- publishes the property on a body tool, and on a tool whose request is a schema reference;
- does not publish it on a tool with no body;
- the instructions name the staging URL on the configured API, the body shape and the limits.

**End to end**, against the real API and the real tool-server token
([13-testing-e2e](13-testing-e2e.md)):
- a script stages a body with no token, and a tool call uses the id twice, once with the
  tenant argument;
- an unknown or expired id answers 404 with the message; a mixed body answers 400;
- the stage endpoint refuses a non-object body and a payload over the limit;
- a staged body cannot move a write into another tenant: the token picks the tenant;
- a web app token gets no swap: the id reaches the use case as an unknown field.

**Guards**: the table has a TTL attribute and no point-in-time recovery; the edge rule
exists, is scoped to the stage path, and is ordered before the general anonymous rule; a
guard test pins the list of endpoints that accept no token, and the stage endpoint is on
it ([18-security-and-privacy](18-security-and-privacy.md) § 2).

---

## 8. Swap points

- **Store:** any key/value store with TTL, or a relational table with an `expires_at`
  column and a scheduled delete. The read-time expiry check stays either way.
- **Edge limiter:** any rate limiter keyed on client IP and scoped to the path.
- **Tool protocol:** MCP or any other. The swap lives in the API, so the tool layer only
  publishes the property and the instructions.
- **Lifetime and size:** tune them, and put the new worst-case cost in the decision record.

**Not swap points:** server-made ids, no read-back endpoint, the swap only for the tool
server's audience, the swap before tenant resolution and the role gate, the expiry check on
read, and a ceiling on every cost axis.

---

## 9. Adopting it

1. Write the decision record ([20-decision-records](20-decision-records.md)) with the
   worst-case cost numbers.
2. Add the table in IaC: global, UUID key, TTL, write cap, no point-in-time recovery.
3. Add the repository port, its in-memory fake and its real-store implementation.
4. Add the stage use case: anonymous role, no personal data.
5. Add the swap function, and call it in the entrypoint straight after body parsing.
6. Add the edge rate rule ahead of the general anonymous rule.
7. In the tool server, add the property to every body tool and the paragraph to the
   instructions.
8. Write the tests in § 7, and see each one fail once for the right reason.
