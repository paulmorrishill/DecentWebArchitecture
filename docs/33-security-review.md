# 33 — Security review

A security review checks the built system against a published standard, item by item, and
records what was found and where the proof is. This file says how to run one on a project
built to this set. The checklists are in [security-review/](security-review/):

| Checklist | Standard | Use it for |
|---|---|---|
| [asvs-5.0](security-review/asvs-5.0.md) | OWASP ASVS 5.0.0, L1 / L2 / L3 | The application: API, web client, sign-in, tokens, data |
| [owasp-top-10](security-review/owasp-top-10.md) | OWASP Top 10, API Security Top 10, LLM Top 10, Agentic Top 10 | Awareness, and a cross-check that the ASVS pass missed no whole class |
| [attacks](security-review/attacks.md) | Named attacks, with OWASP WSTG test ids | The test plan for a manual pass or a penetration test |
| [masvs](security-review/masvs.md) | OWASP MASVS 2 | The mobile app |
| [cyber-essentials](security-review/cyber-essentials.md) | UK Cyber Essentials | The organisation's devices, accounts and cloud estate |
| [iso-27001](security-review/iso-27001.md) | ISO/IEC 27001:2022 Annex A | The information security management system around the product |

**Every row in every checklist says whether a rule in this set covers it**, and links the
rule. That column answers "what does building to this set already give us", not "have we
passed". See § 1, rule 3.

---

## 1. The rules

1. **Pick the ASVS level before the first feature, and record it in a decision record**
   ([20-decision-records](20-decision-records.md)). § 2 says how. The level decides which
   rows apply, and a level picked after the review is a level picked to fit the result.
2. **A review result is evidence, not an opinion.** Every row gets a result — pass, fail,
   not applicable — and a link to the thing that proves it: a test, a guard test, a file
   and line, a screenshot of a setting, a tool report. A row with no evidence is a fail.
3. **"Covered by a rule" is not "passed".** The column says a rule here meets the
   requirement *if it was followed*. The reviewer still checks the code, the configuration
   and the deployed environment. A covered row is the cheapest row to pass, not a row to
   skip.
4. **Every gap ends in one of three things**, written down: a rule added to the project's
   own instructions, a work item with an owner, or a decision record that accepts the risk
   and names who accepted it. A gap that ends in none of them is a gap nobody owns.
5. **A partial row is a gap until the missing part is named.** Write the missing part in
   the evidence column, or the next review reads the rule and marks it covered.
6. **Review the deployed system, not the repository alone.** Half of the controls in
   [29-hardening-that-costs-nothing](29-hardening-that-costs-nothing.md) are settings, and
   a setting can be off in one environment and on in another. Check the environment the
   result is claimed for.
7. **Test against a pre-production environment with production's configuration, never
   against production.** A penetration test that writes, floods or locks accounts on
   production is an incident. Production gets the read-only checks: headers, TLS, DNS
   records, the public surface.
8. **Record the result in the tracker, not in this set.** Each checklist has a CSV with
   empty result, evidence, owner and date columns. Copy it into the project; do not edit the
   checklist page to record a result.
9. **When a review finds a defect a rule should have caught, add the rule.** Add it to the
   project's pitfalls file ([22-pitfalls-checklist](22-pitfalls-checklist.md)) and, if it is
   general, to this set. Then fix the coverage column of every checklist row it covers.

### 1.1 When to review

| Trigger | What to run |
|---|---|
| Before the first production release | ASVS at the chosen level, the Top 10 lists, the attack plan, MASVS if there is an app |
| A change to sign-in, tokens, the role gate, tenancy, the tool server or the OAuth broker | The ASVS chapters for that area (V6–V10, V8 for tenancy), and the attacks that target it |
| A new external input: a webhook, an upload, inbound mail, a new third-party API | V1, V2, V4, V5, V13.2 and the injection, upload and SSRF attacks |
| Every year | All of it. Cyber Essentials must be renewed every 12 months anyway |
| An ISO 27001 surveillance or recertification audit | The ISO checklist, with the ASVS result as the evidence for 8.25–8.29 |

---

## 2. Picking the ASVS level

ASVS 5.0 has three levels. Each one includes the one below it.

| Level | Requirements | For |
|---|---|---|
| L1 | 70 | The minimum for anything on the internet. Mostly testable from outside |
| L2 | 253 | **The default for an application that holds personal data**, and so the default for an application built to this set, which has a personal-data gate ([18-security-and-privacy](18-security-and-privacy.md) § 3) |
| L3 | 345 | Health, financial, safety or legal data, critical infrastructure, or a customer whose contract demands it |

Rules for the choice:

- **Choose L2 unless there is a reason not to, and write the reason down.** L1 is for a
  system that holds nothing about anyone.
- **A level is for the whole application**, not per endpoint. If one area needs L3 — the
  OAuth broker, for example — apply the L3 rows of that chapter as well and say so in the
  record.
- **Mark the rows that do not apply as not applicable, with the reason**: there is no
  WebRTC, there is no GraphQL. The checklist already marks the surfaces this set does not
  have, but a project may add one.

---

## 3. Threat model first

**Before the checklist pass, draw the trust boundaries and list the threats at each
one.** A checklist finds the controls that are missing. It does not find the feature that
should not exist, and it does not know which boundary matters most for this product.

Use STRIDE (spoofing, tampering, repudiation, information disclosure, denial of service,
elevation of privilege) at each boundary. The boundaries in this architecture are:

| Boundary | Where it is set out | The threat to start with |
|---|---|---|
| Internet → edge and gateway | [29](29-hardening-that-costs-nothing.md) § 1, [18](18-security-and-privacy.md) § 1 | A path that the gateway does not authenticate |
| Gateway → dispatcher → use case | [07](07-composition-and-entrypoints.md), [18](18-security-and-privacy.md) § 2 | A stale route or privacy table, which fails open |
| Use case → store | [05](05-repositories-and-persistence.md), [08](08-branches-and-data-scoping.md) | A query without its scope |
| Tenant → tenant | [31](31-multi-tenancy.md) § 2 | A barrier that is only code |
| Machine client → tool server and OAuth broker | [27](27-tool-servers-and-mcp.md) | A confused deputy, and prompt injection through tool output |
| Anonymous staging of large payloads | [26](26-staged-tool-payloads.md) | Cost abuse and a payload read by the wrong caller |
| Third party → webhook, inbound mail, upload | [18](18-security-and-privacy.md) § 4 | An unverified signature, a hostile file |
| Mobile device → offline store and sync | [10](10-mobile.md) | A lost device, and a replayed or forged sync |
| Developer, agent and pipeline → cloud account | [28](28-supply-chain-and-build-inputs.md), [16](16-ci-cd.md), [15](15-infrastructure-and-environments.md) | A dependency or a pull request that runs code with the deploy role |

Keep the model as a file in the repository, next to the decision records, and change it in
the same pull request as a change that adds a boundary.

---

## 4. The automated checks

Some rows are cheaper to check by a tool on every build than by a person once a year. Run
these in the pipeline ([16-ci-cd](16-ci-cd.md)), and treat a finding as a failing check:

| Check | What it finds | Rule here |
|---|---|---|
| Dependency advisories | Known-vulnerable packages | Yes — [28](28-supply-chain-and-build-inputs.md), [18](18-security-and-privacy.md) § 8 |
| Pinned actions, image digests, binary hashes | A moved tag, a swapped download | Yes — [28](28-supply-chain-and-build-inputs.md) |
| Guard tests over the infrastructure code | A free hardening setting that came off | Yes — [29](29-hardening-that-costs-nothing.md) § 6, [12](12-testing-integration-and-guards.md) |
| Secret scanning, with push protection on the host | A committed key | **No** — [18](18-security-and-privacy.md) § 6 forbids the commit but nothing detects it |
| Static analysis (SAST) | Injection, unsafe APIs, unsafe HTML in the clients | **No** |
| Infrastructure-code scanning | Public resources, wide policies, missing encryption beyond the guard tests | Partial — the guard tests of [29](29-hardening-that-costs-nothing.md) § 6 cover the named settings only |
| A baseline dynamic scan (DAST) of the pre-production environment | Missing headers, cookies, exposed files, error pages | **No** |
| A software bill of materials (CycloneDX or SPDX) per release | What shipped, for the next advisory | **No** |

---

## 5. The manual pass and the penetration test

- **Plan the test from [attacks](security-review/attacks.md), not from the tester's
  habits.** Each row names the attack, the OWASP Web Security Testing Guide id, and what to
  try against this architecture.
- **Give the tester one account per role, in two tenants and two branches**, seeded the
  way the end-to-end tests seed ([13-testing-e2e](13-testing-e2e.md)). Most of the
  findings that matter in this architecture are cross-tenant and cross-role, and a tester
  with one account cannot find them.
- **Write the rules of engagement down before the test**: the environment, the dates, the
  accounts, what is out of bounds, who to call when something breaks. Check the cloud
  provider's penetration-testing policy for the services in scope.
- **Use a tester with an accreditation the customer recognises** when the report goes to a
  customer. In the UK that is usually CREST, or CHECK for government work.
- **Every finding becomes a row in the tracker**, mapped to the ASVS requirement it
  breaks, so the next review checks it again.

---

## 6. Coverage of this set

The counts of rows each checklist marks covered, partly covered, and not covered by a rule
in this set. The detail, and the rule for each row, is in the checklist.

| Checklist | ✅ Covered | 🟡 Partial | ❌ Gap | ➖ N/A | 🏢 Org | Total |
|---|---|---|---|---|---|---|
| [ASVS 5.0 L1](security-review/asvs-5.0.md) | 25 | 32 | 12 | 1 | 0 | 70 |
| [ASVS 5.0 L2](security-review/asvs-5.0.md) | 54 | 114 | 61 | 24 | 0 | 253 |
| [ASVS 5.0 L3](security-review/asvs-5.0.md) | 58 | 143 | 110 | 34 | 0 | 345 |
| [Cyber Essentials v3.3](security-review/cyber-essentials.md) | 5 | 27 | 10 | 0 | 8 | 50 |
| [ISO/IEC 27001:2022 Annex A](security-review/iso-27001.md) | 18 | 38 | 6 | 0 | 31 | 93 |
| [OWASP Top 10:2025](security-review/owasp-top-10.md) | 4 | 6 | 0 | 0 | 0 | 10 |
| [OWASP API Security Top 10:2023](security-review/owasp-top-10.md) | 4 | 5 | 1 | 0 | 0 | 10 |
| [OWASP Top 10 for LLM Applications 2026](security-review/owasp-top-10.md) | 5 | 2 | 0 | 3 | 0 | 10 |
| [OWASP Top 10 for LLM Applications 2025](security-review/owasp-top-10.md) | 5 | 2 | 0 | 3 | 0 | 10 |
| [OWASP Top 10 for Agentic Applications (for 2026)](security-review/owasp-top-10.md) | 3 | 5 | 0 | 2 | 0 | 10 |
| [OWASP MASVS 2.1.0](security-review/masvs.md) | 3 | 8 | 10 | 3 | 0 | 24 |
| [Attack catalogue](security-review/attacks.md) | 21 | 21 | 8 | 2 | 0 | 52 |

### 6.1 The gaps that matter most

The gaps below are the ones every review of a project built to this set will find, because
no rule here asks for them yet. Close them with a rule in the project before the review, or
expect them as findings.

1. **Server-side request forgery.** No rule governs an outbound request built from caller
   data: no destination allowlist, no refusal of internal addresses, no rule on following
   redirects. ASVS V1.3.6, V13.2.4–5, V15.3.2; API7:2023; part of A01:2025.
2. **Cross-site scripting in the web client.** No rule bans raw-HTML sinks or requires
   sanitising HTML, and no content security policy is required on the application — only on
   the consent page of [27](27-tool-servers-and-mcp.md) § 6.3. With the token in browser
   storage ([18](18-security-and-privacy.md) § 1), one script injection reads it. ASVS V1.3.1,
   V3.2.2, V3.3.4, V3.4.3.
3. **Multi-factor authentication is offered, not required** — not for the product's own
   administrators, and not for the staff accounts on the cloud console, the git host, the
   pipeline, the app stores and the package registries. This is an automatic Cyber
   Essentials fail. ASVS V6.3.3; CE-UAC-4.
4. **Sessions do not end.** No logout rule that revokes the refresh token, no revocation
   when an account is disabled or a role removed, no idle timeout, no refresh-token rotation
   for the public clients, no re-authentication before a sensitive change. ASVS V7.3–V7.5,
   V10.4.5.
5. **No security audit trail.** Sign-ins, failed authorization, role and membership changes
   and administrative actions are not logged as security events;
   [17](17-observability-and-errors.md) § 8 rightly keeps 401 and 403 out of the error
   report, but nothing records them anywhere else. Nothing protects the logs from change. ASVS
   V16.3.1–2, V16.4.2; A09:2025; ISO 8.15.
6. **No deadline for fixing a known vulnerability.** [28](28-supply-chain-and-build-inputs.md)
   § 4 stops new advisories from entering, but sets no time to fix one already present, and
   nothing retires an end-of-life runtime or image. Cyber Essentials needs 14 days for a
   critical or high fix. CE-PATCH-2, CE-PATCH-4; ASVS V15.2.1.
7. **Backend traffic is not required to use TLS.** The rules stop at the edge. ASVS V12.3.
8. **The mobile app's platform controls.** No rule on cleartext traffic, verified app links
   and deep-link input, WebViews, hiding sensitive screens, the minimum OS version, or
   encrypting the offline database. MASVS-NETWORK, MASVS-PLATFORM, MASVS-STORAGE-1.
9. **Uploads beyond the signed link.** No content check, no malware scan, no archive limits,
   and nothing forces a download rather than a render for user content. ASVS V5.2–V5.4.
10. **Incident response and recovery.** No runbook, no evidence handling, no restore test.
    ISO 5.24–5.28, 8.13; [§ 7](#7-other-things-worth-doing).

---

## 7. Other things worth doing

Not every useful control is in one of the checklists. These are the ones a review of this
architecture should also check, and whether a rule here covers them.

| Item | Why | Rule here |
|---|---|---|
| A `/.well-known/security.txt` (RFC 9116) and a vulnerability disclosure policy | A researcher who finds a hole needs somewhere to send it | **No** |
| A data protection impact assessment and a record of processing (UK GDPR Art. 30 and 35) | Required before high-risk processing of personal data, and the first thing a regulator asks for | Partial — the personal-data gate of [18](18-security-and-privacy.md) § 3 tells you which endpoints are in scope |
| An incident response runbook, with a named owner and a tested contact list | A breach of personal data must be reported to the ICO within 72 hours of becoming aware of it | **No** |
| A restore test from backup, on a schedule | A backup that was never restored is a hope | **No** — [29](29-hardening-that-costs-nothing.md) § 3 and § 7 name recovery, but nothing tests a restore |
| A security audit trail: sign-in, role and membership changes, administrative actions, cross-tenant overrides | ASVS V16.3, ISO 8.15, and the first question after an incident | Partial — [31](31-multi-tenancy.md) § 4.1 names a platform audit log; [17](17-observability-and-errors.md) does not log security events |
| The cloud provider's account benchmark (CIS Foundations Benchmark) and its threat-detection service | Account-level settings no application rule reaches: root account, account trail, detection | Partial — [29](29-hardening-that-costs-nothing.md) § 5 and [15](15-infrastructure-and-environments.md) cover the account split and the roles |
| Build provenance (SLSA) and the OpenSSF Scorecard on the repository | Proof that the artifact came from this source and this pipeline | Partial — [28](28-supply-chain-and-build-inputs.md) pins the inputs but does not attest the output |
| The NCSC Cloud Security Principles, for a UK public-sector buyer | The 14 principles a public-sector buyer assesses a SaaS against | Not mapped here |
| SOC 2 Type II, for a US buyer | The US equivalent of an ISO 27001 certificate in a procurement questionnaire | Not mapped here; most criteria map to ISO 27001 Annex A |

---

## 8. Keeping the checklists current

- **The checklists name the standard's version.** When a new version is published, map it
  again; do not edit the old rows to fit.
- **When a rule in this set changes, grep the checklists for its file and section number**
  and fix the coverage column of every row that cites it.
- **The standard's text in the ASVS checklist is OWASP's, licensed CC BY-SA 4.0.** The
  other checklists are written in this set's own words and use only the identifiers and
  titles of the standards they map.
