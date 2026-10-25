# 29 — Hardening that costs nothing

Most of the controls in [18-security-and-privacy](18-security-and-privacy.md) are
**settings on a service the project already pays for**. They are not bought, and they are
not a project of their own. A team that says "we will do security later" usually means
"we have not turned these on".

This file lists them, says what each one gives, and says what each one takes away. It
also names, at the end, the controls that **do** cost money — so nobody plans a budget
around a free item that is not one.

The service names are the ones Reference A uses. The swap point is the provider; the
shape of each control is the same everywhere.

---

## 1. The edge

### 1.1 The flat-rate plan is the big one

A content delivery network with a flat-rate pricing plan covers **one distribution and
one apex domain for a fixed monthly price**. The price includes the firewall access
control list, its rules, the firewall request fees, the delivery requests and bytes, and
the hosted zone. There is no overage charge.

Reference A measured the pay-as-you-go alternative in October 2026: about **$5 per access
control list per month plus about $1 per rule per month**, as **fixed** charges — the
request volume added almost nothing. Six lists and about forty rules came to about $70 a
month on production, and the staging equivalent tripped the cost kill switch.

| Tier | Price for each distribution | Firewall rules | Cache behaviours | Custom firewall response | Header match | Custom response-headers policy | Access and firewall logs |
|---|---|---|---|---|---|---|---|
| Free | **0** | 5 | 5 | no | no | no | no |
| Paid (Reference A used the first paid tier) | $15 | 25 | 10 | yes | yes | no | yes |

**So a small site gets a real firewall for nothing.** Reference A's free-tier list holds
five rules: two scanner rules, one per-address rate rule, and two managed rule groups.
That is the whole budget, and it covers the common cases.

What the free tier takes away, and you must decide you can live with it:

- **No access log and no firewall log.** You cannot see what was blocked, and anything
  downstream that reads those logs — an address blocker, for example — stops being fed.
- **No custom firewall response.** A blocked request gets the generic 403.
- **No custom response-headers policy** (§ 1.2 is the way round it, with a limit).
- **Five cache behaviours and five rules is a design constraint**, not a starting point.
  Count both before you design around the plan
  ([15-infrastructure-and-environments](15-infrastructure-and-environments.md) § 8).
- The provider **refuses** some firewall features on a plan list, and a subscription fails
  while a refused feature is still attached. Expect to apply the removal first and
  subscribe second.
- The infrastructure tool may have **no resource for the subscription**. Reference A
  subscribes each distribution by hand in the console, after the apply. Write that in the
  decision record, or the next person will look for it in the code
  ([20-decision-records](20-decision-records.md)).

### 1.2 The rest of the edge, all free

| Free item | What it gives |
|---|---|
| Managed security-headers policy | Transport security, content-type sniffing off, frame refusal, referrer policy — on every response, with no policy to write |
| Minimum TLS version setting | Old protocol versions are refused at the edge |
| HTTP to HTTPS redirect | No plaintext request reaches an origin |
| Managed certificate | Issued and renewed with no job to run |
| Origin access control plus "block all public access" on the bucket | The static files have **no** public path at all. Only the one distribution can read them |
| Managed cache and origin-request policies | Needed anyway on a plan, which refuses custom ones |
| Gateway stage throttling | A second rate limit behind the firewall, set as a number |
| Reserved concurrency of zero on a function | The pause lever for a cost emergency |

**Two warnings, both of which have cost a day:**

- **A viewer-response function does not run on a response of 400 or more.** Any header you
  add in such a function — a content security policy, a permissions policy, cross-origin
  headers — is missing from exactly the responses an attacker is probing. On a plan, where
  no custom response-headers policy is allowed, there is no way to add them. Know which
  headers that leaves on an error response, and write it down.
- **A function at the edge bills per request.** It is cheap, not free. The managed policy
  is free; the function is not.

---

## 2. Identity

Every item here is configuration on the managed identity service.

- **The hosted sign-in page**, with the authorization-code flow and PKCE. The application
  never sees the password.
- **Time-based one-time-code second factor.** Free. A second factor sent by text message
  is not — it bills per message.
- **No open sign-up.** Only an administrator creates an account; a new member gets a
  single-use activation link, stored hashed, that expires and cannot be used twice.
- **Token lifetime and refresh-token lifetime** are numbers, not work.
- **Verify the token at the gateway, before any code runs**: signature, issuer, and
  audience against the application's own clients. A token from any other system is
  refused with no code involved.
- **The same answer for every address.** "Send my activation email again" answers
  identically whether or not the address exists, so the platform cannot be used to find
  out who is a member. This is code, and it is free.
- **Password policy by length, not by character classes.**

---

## 3. Data

| Free | Not free (§ 7) |
|---|---|
| Block all public access on every bucket | Point-in-time recovery |
| Default encryption with a provider-owned key | A customer-managed key |
| Deletion protection on every table | Versioning and backup storage |
| A TTL attribute instead of a cleanup job — the expiry deletes cost nothing | Cross-account or cross-region copies |
| Short-lived signed upload and download links | |
| Checking type and size **before** issuing an upload link | |
| Storing a person's id on a record, never a copy of their name or email | |

Two rules that are free and that carry most of the weight:

- **Nothing is reached except through a signed link with a short life.** Reference A uses
  5 to 60 minutes.
- **A TTL is lazy.** The provider may delete up to two days late, so **the reader checks
  the expiry on every read**. "The row exists" is not "the row is valid"
  ([26-staged-tool-payloads](26-staged-tool-payloads.md) § 2.1).

---

## 4. Email

SPF, DKIM, and a DMARC policy of `p=reject` are **DNS records**. The hosted zone is
already paid for, so the authentication of every message the platform sends costs nothing,
and mail that claims to come from the platform but does not is rejected by the receiver.

Restrict the sending identity as well: the API may send only from the platform's own
sender domains, and the service refuses any other From address.

---

## 5. The account and the pipeline

| Free item | What it gives |
|---|---|
| One account per environment | A fault or a credential in staging cannot reach production data |
| Federated short-lived pipeline credentials | No long-lived key is stored anywhere ([28-supply-chain-and-build-inputs](28-supply-chain-and-build-inputs.md) § 7) |
| A role for each function, with grants naming the exact bucket prefix, queue, topic or pool | A compromise of one function reaches one function's resources |
| Inline policies | They do not consume the managed-policy quota per role, which is small — know the number ([15-infrastructure-and-environments](15-infrastructure-and-environments.md) § 4.2) |
| Retention set on every log group | An unbounded log group is the most common surprise line on a bill, **and** an unbounded copy of whatever the logs hold |
| A budget alert per environment | The first few budgets are free |
| Everything created by infrastructure code, applied only by the pipeline | Nobody changes production by hand, and the change has a review |
| The automated dependency updater | Free, and § 3 of [28-supply-chain-and-build-inputs](28-supply-chain-and-build-inputs.md) makes it safe |

---

## 6. The tests that keep them on

A setting that nobody checks is a setting that comes off in a refactor. Each of these is
a guard test over the infrastructure code
([12-testing-integration-and-guards](12-testing-integration-and-guards.md)) and costs
nothing to run:

| Test | Fails when |
|---|---|
| Table protection | A table lacks deletion protection, or lacks recovery where recovery is required |
| Plan-tier guard | A distribution on a plan declares a feature its tier refuses |
| Security headers | A distribution lacks the security-headers policy |
| Scanner block | The firewall scanner rules are missing or out of order |
| Public bucket | A bucket allows public access |
| Role trust | A role trusts a subject with no branch, or a read-only role can write |
| Log retention | A log group has no retention |

Each one is the same shape: read the infrastructure file, assert the setting
([12-testing-integration-and-guards](12-testing-integration-and-guards.md)).

---

## 7. What is NOT free

Say these out loud in the decision record, so a later reader does not assume the whole
list above was free:

- **Firewall charges off a plan**: a fixed charge per access control list and per rule,
  per month, plus a charge for each managed rule group per list.
- **Firewall logs and access logs**: ingestion and storage.
- **Point-in-time recovery**: priced on the stored size.
- **Customer-managed keys**: a monthly charge per key, plus per-request charges. Use them
  for the data that needs them — tokens, publishing credentials, imported cookies — and
  the provider-owned key for the rest.
- **Backup storage and versioning**: priced on size and retention.
- **Edge functions**: priced per request.
- **Second factor by text message**: priced per message.
- **The paid plan tier**: a fixed monthly price per distribution, which is what buys back
  the logs, the custom firewall response and the larger rule budget.

---

## 8. Adopting it

1. Turn on the free settings of § 1.2, § 3 and § 5 today. They need no design.
2. Add the guard tests of § 6 in the same change, or the settings come off later.
3. Add the DNS records of § 4.
4. Measure the firewall bill before choosing a plan tier. The decision is "what do the
   logs and the rule budget cost me", and the free tier answers it for a site that needs
   five rules.
5. Write the plan subscription step into the decision record, because the infrastructure
   code cannot do it.
