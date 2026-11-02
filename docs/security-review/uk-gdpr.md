# UK GDPR checklist

The UK GDPR and the Data Protection Act 2018, **as amended by the Data (Use and Access) Act 2025**, whose data protection changes are all in force as of November 2026, and the Privacy and Electronic Communications Regulations 2003 (PECR) as amended by the same Act. The rows follow the ten sections and 77 control measures of the **ICO Accountability toolkit**, part of the ICO's Data Protection Audit Framework, then add the articles a SaaS needs that the toolkit does not list one by one. How to run the review is [33-security-review](../33-security-review.md).

The UK GDPR binds an **organisation**, and most of its duties are documents, contracts and people. An architecture can supply the evidence for data protection by design, security, deletion and isolation. It cannot write the privacy notice, sign the processing contract or report a breach, so expect 🏢 rows.

**Controller or processor.** A B2B SaaS built to this set holds two kinds of personal data, and its duties differ for each:

| Data | The SaaS is | Who answers to the people in it |
|---|---|---|
| The customer's data: everything in the per-customer store and file prefix, and the members' sign-in records as used to run the service ([31](../31-multi-tenancy.md#41-two-kinds-of-database) § 4.1) | **Processor.** It acts on the customer's instructions in the contract | The customer, which is the controller ([31](../31-multi-tenancy.md#61-what-is-per-tenant-and-is-not-a-row) § 6.1). The SaaS supplies the tools |
| Its own data: the customer's contacts, billing, support tickets, the platform audit log, platform security records, its own website, app analytics and marketing lists | **Controller** | The SaaS |

Each item starts with **[C]** (a controller duty, for the SaaS's own data), **[P]** (a processor duty, for customer data) or **[C+P]** (both). The ICO toolkit is written for a controller; its organisational rows apply to a processor too. **A processor that uses customer data for a purpose of its own — analytics, benchmarks, training a model, a support engineer's curiosity — is a controller for that use** (Art 28(10)), with every [C] row that follows.

- **Repo rule.** ✅ a rule here, if followed, meets the item. 🟡 a rule meets part of it. ❌ no rule. ➖ the architecture as written has no such surface. 🏢 an organisational, people or physical control that an architecture cannot cover. **A tick is not a pass.** It says where to look; the reviewer still checks.
- **Tracker.** [uk-gdpr.csv](uk-gdpr.csv) has the same rows with empty result columns.
- **Wording.** Items are paraphrased; the legislation and the regulator's guidance are the authority. The ICO marks its toolkit and much of its guidance as under review after the Data (Use and Access) Act, so re-check both before relying on a row. **This is not legal advice.**
- **The regulator changed its name.** Since 30 September 2026 the Information Commissioner's functions belong to the **Information Commission**, still known as the ICO. Reports, fees and complaints go where they went before.

---

## What a SaaS team gets wrong

- **The breach clock is the customer's, and you spend it.** The controller has 72 hours from becoming aware to report to the Information Commission. As processor you must tell the customer "without undue delay", and every hour you take is gone from their 72. Write the runbook, with a breach contact per customer, before the first customer signs. No rule here has one ([33](../33-security-review.md#7-other-things-worth-doing) § 7). Log every breach, including the ones you decide not to report (Art 33(5)).
- **Rights are per person; this set's tools are per customer.** [31](../31-multi-tenancy.md#8-lifecycle-r26-to-r30) § 8 exports and deletes a whole customer. The customer will ask you to find, export, correct, restrict or erase *one member*. Erasure must reach backups (or let them age out on a stated schedule), logs, the error tracker, the search index and every phone's offline store. Restriction has no mechanism at all.
- **Error reports carry personal data even when the tags do not.** [17](../17-observability-and-errors.md#1-one-reporter-everywhere) § 1 keeps it out of the context, but exception messages, breadcrumbs, request bodies, IP addresses and session replays are the SDK's defaults. The error tracker and the crash reporter are sub-processors, usually in the US: list them, sign their DPA, and turn the defaults off.
- **Browser storage is a cookie in law.** PECR covers any storage on or reading from the device: local storage, app storage and SDK identifiers as well as cookies. The token in browser storage is strictly necessary. An analytics identifier fits the new statistical exception only for that sole purpose, with clear information and a free way to object; an advertising or cross-site identifier needs consent. PECR fines now reach UK GDPR levels.
- **The OAuth consent page is not GDPR consent.** [27](../27-tool-servers-and-mcp.md#63-the-consent-page) § 6.3 stops a confused deputy. Employee and member data rarely rests on consent at all, because a worker cannot freely refuse; use contract or legitimate interests, and write the assessment.
- **Recognised legitimate interests are a short, closed list.** The new basis in Annex 1 covers requests from public bodies, national and public security, emergencies, crime and safeguarding. Direct marketing, intra-group administration and network security are named as legitimate interests too, but they still need the balancing test.
- **Background location is tracking.** The ICO's list of processing that needs a DPIA includes tracking a person's location or behaviour, and a field app that records location ([32](../32-mobile-delivery.md#91-store-review) § 9.1) does that. Screen it before the first release, and decide whether the scale makes a DPO mandatory.
- **A transfer is wherever the data or the support engineer is.** No rule here pins the region of a store, a log or a vendor. Since the DUAA the test is that protection abroad is "not materially lower" than in the UK, and the ICO's transfer guidance was rewritten in January 2026. Existing IDTAs and UK Addenda still work, and the UK-US data bridge covers only US companies certified under the Data Privacy Framework.
- **Two duties are new in 2026.** Subject access needs only a reasonable and proportionate search, and the clock stops while you wait for a clarification you reasonably need — but you must show what you searched. Since 19 June 2026 every controller must make complaining easy, acknowledge a complaint within 30 days and report the outcome without undue delay.

## Coverage

| | ✅ Covered | 🟡 Partial | ❌ Gap | ➖ N/A | 🏢 Org | Total |
|---|---|---|---|---|---|---|
| Leadership and oversight | 0 | 0 | 0 | 0 | 6 | 6 |
| Policies and procedures | 0 | 3 | 0 | 0 | 1 | 4 |
| Training and awareness | 0 | 0 | 0 | 0 | 5 | 5 |
| Individuals' rights | 0 | 4 | 1 | 1 | 5 | 11 |
| Transparency | 0 | 2 | 1 | 1 | 3 | 7 |
| Records of processing and lawful basis | 0 | 2 | 4 | 3 | 1 | 10 |
| Contracts and data sharing | 0 | 4 | 2 | 0 | 3 | 9 |
| Risks and DPIAs | 1 | 3 | 1 | 0 | 0 | 5 |
| Records management and security | 0 | 11 | 0 | 0 | 1 | 12 |
| Breach response and monitoring | 0 | 4 | 2 | 0 | 2 | 8 |
| Principles and security of processing | 0 | 4 | 0 | 0 | 1 | 5 |
| Processor duties | 2 | 4 | 2 | 0 | 1 | 9 |
| International transfers | 0 | 1 | 2 | 0 | 2 | 5 |
| PECR: device storage and electronic marketing | 0 | 3 | 2 | 1 | 0 | 6 |
| **All** | 3 | 45 | 17 | 6 | 31 | 102 |

## Leadership and oversight

| ID | Item | Repo rule | Rules | Note |
|---|---|---|---|---|
| GDPR-LEAD-1 | [C+P] A structure for managing data protection, with a senior person accountable | 🏢 Org |  | Organisation chart and terms of reference; nothing an architecture sets. |
| GDPR-LEAD-2 | [C+P] A DPO where Art 37 requires one (large-scale systematic monitoring or special category data), and supported | 🏢 Org |  | Background location tracking (32 §9.1) at scale can make a DPO mandatory; decide and record. |
| GDPR-LEAD-3 | [C+P] The DPO is independent, has no conflicting role and reports to the top | 🏢 Org |  | The DPO must not also be the person deciding the processing, such as the CTO. |
| GDPR-LEAD-4 | [C+P] Operational roles carry out data protection day to day | 🏢 Org |  | Named owners exist for packages (28 §1.2), not for personal data. |
| GDPR-LEAD-5 | [C+P] An oversight group for data protection and information governance | 🏢 Org |  |  |
| GDPR-LEAD-6 | [C+P] Operational-level groups feed the oversight group | 🏢 Org |  |  |

## Policies and procedures

| ID | Item | Repo rule | Rules | Note |
|---|---|---|---|---|
| GDPR-POL-1 | [C+P] Policies and procedures tell staff enough to meet their data protection duties | 🟡 Partial | [18 §3](../18-security-and-privacy.md#3-personal-data), [18 §7](../18-security-and-privacy.md#7-operating-on-production), [23 §0.2](../23-new-feature-recipe.md#02-harden-the-specification) | Engineering rules only; no data protection policy for support, sales or operations staff. |
| GDPR-POL-2 | [C+P] Policies are reviewed and approved so they stay consistent and effective | 🟡 Partial | [20 §3](../20-decision-records.md#3-process), [33 §1](../33-security-review.md#1-the-rules) | Rule changes go through decision records and review; no data protection policy owner or review date. |
| GDPR-POL-3 | [C+P] Staff know the policies and procedures that apply to them | 🏢 Org |  |  |
| GDPR-POL-4 | [C+P] Policies build in data protection by design and by default | 🟡 Partial | [18 §3](../18-security-and-privacy.md#3-personal-data), [23 §0.2](../23-new-feature-recipe.md#02-harden-the-specification), [22 §4](../22-pitfalls-checklist.md#4-data-leaking-to-a-lower-privileged-role) | Every feature spec asks about personal data and deletion; nothing outside engineering. |

## Training and awareness

| ID | Item | Repo rule | Rules | Note |
|---|---|---|---|---|
| GDPR-TRAIN-1 | [C+P] A data protection training programme for all staff | 🏢 Org |  |  |
| GDPR-TRAIN-2 | [C+P] Induction and refresher training for everyone | 🏢 Org |  |  |
| GDPR-TRAIN-3 | [C+P] Specialist training for roles with key data protection duties | 🏢 Org |  | Include support staff who can see customer data, and on-call engineers who handle a breach. |
| GDPR-TRAIN-4 | [C+P] Evidence that staff completed and understood the training | 🏢 Org |  |  |
| GDPR-TRAIN-5 | [C+P] Ongoing awareness across the organisation | 🏢 Org |  |  |

## Individuals' rights

The ICO toolkit calls this section "information rights". For customer data the customer answers the request and the SaaS supplies the tools (GDPR-PROC-4).

| ID | Item | Repo rule | Rules | Note |
|---|---|---|---|---|
| GDPR-RIGHTS-1 | [C] People are told their rights, and staff recognise a request made verbally or in writing | 🏢 Org |  | A request need not name the law or come through a form. |
| GDPR-RIGHTS-2 | [C] Enough resource to handle requests | 🏢 Org |  |  |
| GDPR-RIGHTS-3 | [C] Requests are logged and each one is tracked to closure | 🏢 Org |  | A ticket queue is enough; it is the evidence of the deadline. |
| GDPR-RIGHTS-4 | [C] Access (Art 15) answered in one month, extendable by two; reasonable and proportionate search; clock stops while clarification is awaited (Art 12A) | 🟡 Partial | [29 §3](../29-hardening-that-costs-nothing.md#3-data), [18 §3](../18-security-and-privacy.md#3-personal-data), [31 R5](../31-multi-tenancy.md#keys) | Person-id references make a search cheap; no rule for finding all of one person's data. |
| GDPR-RIGHTS-5 | [C] Handling of requests is monitored and the results used to improve | 🏢 Org |  |  |
| GDPR-RIGHTS-6 | [C] Rectification (Art 16): wrong data corrected and incomplete data completed | 🟡 Partial | [29 §3](../29-hardening-that-costs-nothing.md#3-data) | One copy of each fact, referenced by id; no rule that a person can correct their own record. |
| GDPR-RIGHTS-7 | [C] Erasure (Art 17) and objection (Art 21): data deleted, suppressed or no longer processed | 🟡 Partial | [31 R30](../31-multi-tenancy.md#lifecycle), [31 §8](../31-multi-tenancy.md#8-lifecycle-r26-to-r30), [29 §3](../29-hardening-that-costs-nothing.md#3-data), [23 §0.2](../23-new-feature-recipe.md#02-harden-the-specification) | Deletion list is per customer, not per person; backups, logs, error tracker and devices not covered. |
| GDPR-RIGHTS-8 | [C] Restriction (Art 18): processing of a person's data can be restricted | ❌ Gap | [31 R29](../31-multi-tenancy.md#lifecycle) | Read-only exists for a whole customer; nothing marks one person's records restricted. |
| GDPR-RIGHTS-9 | [C] Portability (Art 20): a person's data in a structured, machine-readable form | 🟡 Partial | [31 §8](../31-multi-tenancy.md#8-lifecycle-r26-to-r30) | Export is a whole-customer hand-over; no per-person export. |
| GDPR-RIGHTS-10 | [C] Automated decisions (Art 22A-22D): information, representations, human intervention, contest | ➖ N/A | [27 §2.1](../27-tool-servers-and-mcp.md#21-reasons-to-keep-an-endpoint-off-the-list) | No solely automated significant decision as written; a model may not perform a person's legal act. |
| GDPR-RIGHTS-11 | [C] Complaints: an easy route such as an online form, acknowledged within 30 days, outcome without undue delay (DPA 2018 s164A, since 19 June 2026) | 🏢 Org |  | New duty; tell people they can also complain to the Information Commission. |

## Transparency

| ID | Item | Repo rule | Rules | Note |
|---|---|---|---|---|
| GDPR-TRANS-1 | [C] The privacy notice has everything Art 13 and 14 require | 🟡 Partial | [32 §9.1](../32-mobile-delivery.md#91-store-review), [31 §6.1](../31-multi-tenancy.md#61-what-is-per-tenant-and-is-not-a-row) | Notice link in app and store listing, notice per customer; content not specified. |
| GDPR-TRANS-2 | [C] People get the notice at the right time: at collection, or within a month for data from elsewhere | 🟡 Partial | [32 §9.1](../32-mobile-delivery.md#91-store-review), [30 §4](../30-sign-in-branding-and-email.md#4-the-email-text-the-custom-message-trigger) | Disclosure screen before the location prompt; invitation email does not carry the notice. |
| GDPR-TRANS-3 | [C] The notice is concise, clear and in plain language for its audience | 🏢 Org |  |  |
| GDPR-TRANS-4 | [C] Automated decision-making and profiling are explained | ➖ N/A |  | None as written; add a row if a feature scores or ranks people. |
| GDPR-TRANS-5 | [C] Front-line staff can explain the privacy information | 🏢 Org |  |  |
| GDPR-TRANS-6 | [C] The notice is reviewed regularly against what the system actually does | 🏢 Org |  | Trigger a review from the privacy-table diff (06 §6.4). |
| GDPR-TRANS-7 | [C] Tools that let people see and control their data, such as a privacy dashboard | ❌ Gap |  | No rule for a member to see, download or correct their own data. |

## Records of processing and lawful basis

| ID | Item | Repo rule | Rules | Note |
|---|---|---|---|---|
| GDPR-ROPA-1 | [C+P] A data map: what personal data is held and where | 🟡 Partial | [18 §3](../18-security-and-privacy.md#3-personal-data), [31 §4.1](../31-multi-tenancy.md#41-two-kinds-of-database), [31 R30](../31-multi-tenancy.md#lifecycle), [15 §1](../15-infrastructure-and-environments.md#1-layout) | Privacy table lists endpoints, deletion list lists stores; no field-level inventory at rest. |
| GDPR-ROPA-2 | [C+P] A documented, complete and accurate record of processing activities (ROPA) | 🟡 Partial | [18 §3](../18-security-and-privacy.md#3-personal-data), [33 §7](../33-security-review.md#7-other-things-worth-doing) | Inputs exist; no ROPA document. The under-250 exemption does not apply to regular processing. |
| GDPR-ROPA-3 | [C+P] The ROPA holds every Art 30 field: Art 30(1) as controller, Art 30(2) per customer as processor | ❌ Gap |  | Processor record needs each controller, categories of processing, transfers and security measures. |
| GDPR-ROPA-4 | [C+P] The ROPA links to contracts, DPIAs, retention and other records | ❌ Gap | [20 §2](../20-decision-records.md#2-when-to-write-one) | Decision records are the natural link target; no rule asks for it. |
| GDPR-ROPA-5 | [C] Lawful basis documented and justified per purpose (Art 6), including recognised legitimate interests (Art 6(1)(ea), Annex 1) | ❌ Gap | [20 §2](../20-decision-records.md#2-when-to-write-one) | Account and billing: contract. Annex 1 is a closed list; marketing and security still need the balancing test. |
| GDPR-ROPA-6 | [C] Purposes and lawful bases are published | 🏢 Org |  | In the privacy notice. |
| GDPR-ROPA-7 | [C] Where consent is the basis: specific, informed, unbundled, opt-in, recorded and as easy to withdraw | ➖ N/A | [27 §6.3](../27-tool-servers-and-mcp.md#63-the-consent-page) | Nothing relies on consent as written; the OAuth consent page is not Art 7 consent. |
| GDPR-ROPA-8 | [C] Consent records are reviewed and refreshed | ➖ N/A |  |  |
| GDPR-ROPA-9 | [C] Age checks and parental consent where an online service is offered to children (Art 8) | ➖ N/A | [29 §2](../29-hardening-that-costs-nothing.md#2-identity) | No open sign-up; administrators create accounts for adults. |
| GDPR-ROPA-10 | [C] A legitimate interests assessment for each purpose that relies on legitimate interests | ❌ Gap |  | Security logging, error reporting, product analytics and B2B prospecting usually need one. |

## Contracts and data sharing

| ID | Item | Repo rule | Rules | Note |
|---|---|---|---|---|
| GDPR-SHARE-1 | [C] Decisions to share personal data with others are managed | 🟡 Partial | [27 §2](../27-tool-servers-and-mcp.md#2-the-allowlist-lives-in-the-api-not-in-the-tool-server), [18 §10](../18-security-and-privacy.md#10-features-that-hand-the-user-to-a-third-party) | Personal-data endpoints are off the machine-client list by test; no sharing policy. |
| GDPR-SHARE-2 | [C] Data sharing agreements with the controllers data is routinely shared with | 🏢 Org |  |  |
| GDPR-SHARE-3 | [C+P] Restricted transfers follow a procedure | ❌ Gap |  | See the international transfers section. |
| GDPR-SHARE-4 | [C+P] Procedures for the work processors do: hosting, identity, mail, error tracking, crash reports | 🟡 Partial | [17 §1](../17-observability-and-errors.md#1-one-reporter-everywhere), [28 §1](../28-supply-chain-and-build-inputs.md#1-no-new-direct-dependency-without-a-named-decision) | Each vendor is a port with one implementation; no sub-processor register. |
| GDPR-SHARE-5 | [C+P] Every controller-processor contract has the Art 28(3) terms, both with customers and with sub-processors | 🏢 Org |  | Customer DPA and each vendor's DPA; legal, not code. |
| GDPR-SHARE-6 | [C+P] Due diligence that each processor has adequate technical and organisational measures | ❌ Gap | [28 §1.3](../28-supply-chain-and-build-inputs.md#13-what-the-agent-checks-before-it-proposes-a-package) | Packages are vetted; the services that hold the data are not. |
| GDPR-SHARE-7 | [C+P] Processors' compliance with their contracts is reviewed | 🏢 Org |  |  |
| GDPR-SHARE-8 | [C+P] Data protection by design weighs in when choosing services and products | 🟡 Partial | [28 §1](../28-supply-chain-and-build-inputs.md#1-no-new-direct-dependency-without-a-named-decision), [20 §2](../20-decision-records.md#2-when-to-write-one), [29 §7](../29-hardening-that-costs-nothing.md#7-what-is-not-free) | Named decision per dependency; no privacy criteria for a SaaS vendor. |
| GDPR-SHARE-9 | [C+P] Only the personal data needed goes to processors and third parties | 🟡 Partial | [17 §1](../17-observability-and-errors.md#1-one-reporter-everywhere), [09 §10](../09-frontend.md#10-errors-and-observability-in-the-client), [18 §3](../18-security-and-privacy.md#3-personal-data), [27 §2](../27-tool-servers-and-mcp.md#2-the-allowlist-lives-in-the-api-not-in-the-tool-server), [10 §3](../10-mobile.md#3-sync) | No personal data in report context or tags; SDK defaults (IP, breadcrumbs, replay) not addressed. |

## Risks and DPIAs

The DPIA template and the LINDDUN prompts are [below](#dpia-template-and-linddun-prompts).

| ID | Item | Repo rule | Rules | Note |
|---|---|---|---|---|
| GDPR-DPIA-1 | [C+P] Information risks are identified, recorded and managed | 🟡 Partial | [33 §3](../33-security-review.md#3-threat-model-first), [20 §2](../20-decision-records.md#2-when-to-write-one) | Threat model per boundary; no risk register. |
| GDPR-DPIA-2 | [C+P] Data protection by design and by default (Art 25) | ✅ Covered | [18 §3](../18-security-and-privacy.md#3-personal-data), [06 §6.3](../06-api-contract-and-codegen.md#63-therefore-generate-at-deploy-time-and-fail-closed), [29 §3](../29-hardening-that-costs-nothing.md#3-data), [27 §2](../27-tool-servers-and-mcp.md#2-the-allowlist-lives-in-the-api-not-in-the-tool-server), [31 R12](../31-multi-tenancy.md#barriers), [15 §2](../15-infrastructure-and-environments.md#2-environments) | Required privacy flag, fail-closed gate, ids not copies, default-off allowlist, non-code tenant barrier. |
| GDPR-DPIA-3 | [C] Screening decides whether a DPIA is needed, and records the decision | 🟡 Partial | [23 §0.2](../23-new-feature-recipe.md#02-harden-the-specification) | Every feature asks about personal data; no DPIA screening question. |
| GDPR-DPIA-4 | [C] DPIAs hold the Art 35(7) content and are documented | ❌ Gap | [33 §7](../33-security-review.md#7-other-things-worth-doing) | Template below; keep it beside the threat model. |
| GDPR-DPIA-5 | [C] DPIA risks are acted on; high residual risk goes to the Information Commission first (Art 36) | 🟡 Partial | [33 §1](../33-security-review.md#1-the-rules) | Every finding ends in a rule, work item or decision record; nothing on prior consultation. |

## Records management and security

| ID | Item | Repo rule | Rules | Note |
|---|---|---|---|---|
| GDPR-SEC-1 | [C+P] Minimum standards for creating records, and records can be found and retrieved | 🟡 Partial | [31 §4.1](../31-multi-tenancy.md#41-two-kinds-of-database), [08 §1](../08-branches-and-data-scoping.md#1-a-branch-is-not-a-tenant), [29 §3](../29-hardening-that-costs-nothing.md#3-data) | Every row has a tenant and branch in its key; no per-person lookup. |
| GDPR-SEC-2 | [C+P] Personal data protected in transit and when sent to another organisation | 🟡 Partial | [29 §1.2](../29-hardening-that-costs-nothing.md#12-the-rest-of-the-edge-all-free), [29 §4](../29-hardening-that-costs-nothing.md#4-email), [18 §1](../18-security-and-privacy.md#1-authentication) | TLS at the edge, authenticated mail; backend TLS not required (33 §6.1). |
| GDPR-SEC-3 | [C+P] Records are accurate, adequate and not excessive (Art 5(1)(c)-(d)) | 🟡 Partial | [18 §3](../18-security-and-privacy.md#3-personal-data), [29 §3](../29-hardening-that-costs-nothing.md#3-data), [10 §3](../10-mobile.md#3-sync) | Responses minimised and facts held once; no rule against collecting a field nobody needs. |
| GDPR-SEC-4 | [C+P] A retention schedule for all personal data, reviewed regularly (Art 5(1)(e)) | 🟡 Partial | [17 §5](../17-observability-and-errors.md#5-logging), [29 §6](../29-hardening-that-costs-nothing.md#6-the-tests-that-keep-them-on), [29 §3](../29-hardening-that-costs-nothing.md#3-data), [31 §6.1](../31-multi-tenancy.md#61-what-is-per-tenant-and-is-not-a-row), [31 §8](../31-multi-tenancy.md#8-lifecycle-r26-to-r30) | Log retention enforced by test, TTLs; no schedule for domain data, backups, devices, vendors. |
| GDPR-SEC-5 | [C+P] Destruction methods stop disclosure before, during and after disposal | 🟡 Partial | [31 R30](../31-multi-tenancy.md#lifecycle), [18 §6](../18-security-and-privacy.md#6-secrets) | Deletion list includes backups; nothing on device stores or vendor copies. |
| GDPR-SEC-6 | [C+P] An asset register of the systems that store or process personal data | 🟡 Partial | [15 §1](../15-infrastructure-and-environments.md#1-layout), [31 R30](../31-multi-tenancy.md#lifecycle) | Infrastructure code lists cloud resources; vendors and devices missing. |
| GDPR-SEC-7 | [C+P] Acceptable-use rules for software that processes personal data | 🟡 Partial | [28 §1](../28-supply-chain-and-build-inputs.md#1-no-new-direct-dependency-without-a-named-decision), [18 §7](../18-security-and-privacy.md#7-operating-on-production) | Package admission and data-blind production; no general rules. |
| GDPR-SEC-8 | [C+P] Access limited to authorised staff and reviewed regularly | 🟡 Partial | [18 §2](../18-security-and-privacy.md#2-authorization), [18 §7](../18-security-and-privacy.md#7-operating-on-production), [31 R20](../31-multi-tenancy.md#data), [31 §5.3](../31-multi-tenancy.md#53-the-role-split-r20) | Role gate and audited operator role; no access review. |
| GDPR-SEC-9 | [C+P] Unauthorised access prevented: authentication, vulnerability management, malware protection | 🟡 Partial | [18 §1](../18-security-and-privacy.md#1-authentication), [29 §2](../29-hardening-that-costs-nothing.md#2-identity), [28 §4](../28-supply-chain-and-build-inputs.md#4-advisories-and-where-a-package-came-from) | MFA offered not required; no fix deadline (33 §6.1). |
| GDPR-SEC-10 | [C+P] Risks of mobile devices, remote working and removable media managed | 🟡 Partial | [10 §6](../10-mobile.md#6-authentication) | Tokens in secure storage; the offline database is not encrypted, staff devices out of scope. |
| GDPR-SEC-11 | [C+P] Physical locations secured | 🏢 Org |  | Cloud provider for the data centre; offices are the organisation's. |
| GDPR-SEC-12 | [C+P] Continuity plans and backups protect against loss of personal data | 🟡 Partial | [29 §3](../29-hardening-that-costs-nothing.md#3-data), [29 §7](../29-hardening-that-costs-nothing.md#7-what-is-not-free) | Deletion protection free, recovery paid; no restore test (33 §7). |

## Breach response and monitoring

| ID | Item | Repo rule | Rules | Note |
|---|---|---|---|---|
| GDPR-BREACH-1 | [C+P] Incidents and personal data breaches are detected, managed and recorded | 🟡 Partial | [17 §1](../17-observability-and-errors.md#1-one-reporter-everywhere), [17 §2](../17-observability-and-errors.md#2-silent-recovery-is-banned), [31 §5.3](../31-multi-tenancy.md#53-the-role-split-r20) | Errors reported, operator actions audited; no security audit trail (33 §6.1). |
| GDPR-BREACH-2 | [C] Breaches are assessed and, unless unlikely to risk people, reported to the Information Commission within 72 hours (Art 33) | ❌ Gap | [33 §7](../33-security-review.md#7-other-things-worth-doing) | No runbook, owner or contact list. |
| GDPR-BREACH-3 | [C] People are told without undue delay when a breach is likely to put them at high risk (Art 34) | ❌ Gap |  | Needs a contact route per customer and a message template. |
| GDPR-BREACH-4 | [C+P] Every breach, reported or not, is logged with facts, effects and remedy (Art 33(5)), and reviewed | 🟡 Partial | [20 §2](../20-decision-records.md#2-when-to-write-one), [33 §1](../33-security-review.md#1-the-rules) | Lessons become rules and decision records; no breach log. |
| GDPR-BREACH-5 | [C+P] External audits or other independent compliance checks | 🟡 Partial | [33 §5](../33-security-review.md#5-the-manual-pass-and-the-penetration-test) | Security testing only; no data protection audit. |
| GDPR-BREACH-6 | [C+P] An internal audit programme covering data protection | 🟡 Partial | [33 §1.1](../33-security-review.md#11-when-to-review) | Yearly review of all checklists; this one included. |
| GDPR-BREACH-7 | [C+P] Business targets for data protection compliance | 🏢 Org |  |  |
| GDPR-BREACH-8 | [C+P] Monitoring results and management information reach senior management | 🏢 Org |  |  |

## Principles and security of processing

UK GDPR articles the ICO toolkit assumes rather than lists.

| ID | Item | Repo rule | Rules | Note |
|---|---|---|---|---|
| GDPR-PRIN-1 | [C+P] Purpose limitation (Art 5(1)(b), compatibility rules in Art 8A): customer data is not reused for the SaaS's own purposes | 🟡 Partial | [27 §2](../27-tool-servers-and-mcp.md#2-the-allowlist-lives-in-the-api-not-in-the-tool-server), [03 §3.3.2](../03-backend-domain-and-ports.md#332-the-client-instance-identifier) | No rule bans analytics, benchmarks or model training on customer data; doing so makes you a controller (Art 28(10)). |
| GDPR-PRIN-2 | [C+P] Production personal data is never copied to local or pre-production environments | 🟡 Partial | [15 §2](../15-infrastructure-and-environments.md#2-environments), [14 §4](../14-local-dev-and-simulators.md#4-seeding), [32 §14.3](../32-mobile-delivery.md#143-data-on-screen) | Pre-production data is disposable and seeded via the API; no explicit ban on copying production. |
| GDPR-PRIN-3 | [C+P] Accountability (Art 5(2), 24): compliance can be shown with evidence | 🟡 Partial | [33 §1](../33-security-review.md#1-the-rules), [20 §2](../20-decision-records.md#2-when-to-write-one) | Review results carry evidence; no ROPA or DPIA yet. |
| GDPR-PRIN-4 | [C] The data protection fee is paid and the registration kept current | 🏢 Org |  | Every controller pays unless exempt; the processor role does not remove it. |
| GDPR-PRIN-5 | [C+P] Security of processing (Art 32): risk-based measures, encryption, resilience, restore, regular testing | 🟡 Partial | [29 §3](../29-hardening-that-costs-nothing.md#3-data), [31 R12](../31-multi-tenancy.md#barriers), [33 §1.1](../33-security-review.md#11-when-to-review), [29 §7](../29-hardening-that-costs-nothing.md#7-what-is-not-free) | Encryption at rest and yearly testing; no restore test, backend TLS or offline-store encryption. |

## Processor duties

Duties the SaaS has as processor for its customers' data (Art 28(3), 29, 33(2)). The customer will copy these into its due diligence questionnaire.

| ID | Item | Repo rule | Rules | Note |
|---|---|---|---|---|
| GDPR-PROC-1 | [P] Personal data is processed only on the customer's documented instructions, and staff do not browse it (Art 29) | 🟡 Partial | [18 §7](../18-security-and-privacy.md#7-operating-on-production), [31 §5.3](../31-multi-tenancy.md#53-the-role-split-r20), [31 R21](../31-multi-tenancy.md#data) | Production is data-blind and operator reads audited; the instructions are the contract. |
| GDPR-PROC-2 | [P] People who can reach customer data are bound to confidentiality | 🏢 Org |  |  |
| GDPR-PROC-3 | [P] Sub-processors only with the customer's authorisation; changes notified in advance; terms flowed down | ❌ Gap |  | No sub-processor list; identity, mail, hosting, error and crash reporting all qualify. |
| GDPR-PROC-4 | [P] Tools that let the customer answer rights requests: find, export, correct, restrict and erase one person | 🟡 Partial | [31 §8](../31-multi-tenancy.md#8-lifecycle-r26-to-r30), [29 §3](../29-hardening-that-costs-nothing.md#3-data) | Whole-customer export and delete only. |
| GDPR-PROC-5 | [P] The customer gets help with security, breaches, DPIAs and prior consultation (Art 32-36) | 🟡 Partial | [33 §1](../33-security-review.md#1-the-rules), [33 §3](../33-security-review.md#3-threat-model-first) | Checklist results and threat model are the pack; no DPIA input document. |
| GDPR-PROC-6 | [P] At the end of the contract, data is returned and then deleted, copies included | ✅ Covered | [31 R30](../31-multi-tenancy.md#lifecycle), [31 §8](../31-multi-tenancy.md#8-lifecycle-r26-to-r30), [31 R8](../31-multi-tenancy.md#topology) | Hand-over of the customer's database, then the tested deletion list. |
| GDPR-PROC-7 | [P] Information and audits that show compliance are available to the customer | 🟡 Partial | [33 §1](../33-security-review.md#1-the-rules) | Tracker with evidence per row; no customer-facing trust pack. |
| GDPR-PROC-8 | [P] The customer is told of a breach without undue delay (Art 33(2)) | ❌ Gap | [31 §6.1](../31-multi-tenancy.md#61-what-is-per-tenant-and-is-not-a-row) | Controller recorded per customer; no breach contact or procedure. Your delay spends their 72 hours. |
| GDPR-PROC-9 | [P] One customer's data never reaches another customer | ✅ Covered | [31 R12](../31-multi-tenancy.md#barriers), [31 R13](../31-multi-tenancy.md#barriers), [31 R17](../31-multi-tenancy.md#barriers), [31 R18](../31-multi-tenancy.md#data), [31 §9](../31-multi-tenancy.md#9-tests), [08 §8](../08-branches-and-data-scoping.md#8-tests-that-enforce-isolation) | A barrier that is not code, tenant from the verified claim, isolation tests. |

## International transfers

UK GDPR Chapter V as amended by the Data (Use and Access) Act 2025.

| ID | Item | Repo rule | Rules | Note |
|---|---|---|---|---|
| GDPR-XFER-1 | [C+P] Where every store, backup, log and vendor holds personal data, and who can reach it from abroad, is known | ❌ Gap |  | No rule pins a region; vendor support access from abroad is a transfer too. |
| GDPR-XFER-2 | [C+P] Each restricted transfer has a mechanism: UK adequacy regulations (UK-US data bridge for certified US firms), or the IDTA or the EU SCCs with the UK Addendum | 🏢 Org |  | Contract terms; existing IDTAs and Addenda still work, but the ICO plans DUAA updates to both. |
| GDPR-XFER-3 | [C+P] The data protection test is met and recorded: protection not materially lower than in the UK | ❌ Gap | [20 §2](../20-decision-records.md#2-when-to-write-one) | The statutory name for the transfer risk assessment; one decision record per vendor fits. |
| GDPR-XFER-4 | [P] As processor, transfers happen only as the customer's contract allows | 🏢 Org |  |  |
| GDPR-XFER-5 | [C+P] Keys, data-blind operation and minimal vendor payloads limit what a transfer exposes | 🟡 Partial | [29 §7](../29-hardening-that-costs-nothing.md#7-what-is-not-free), [18 §7](../18-security-and-privacy.md#7-operating-on-production), [17 §1](../17-observability-and-errors.md#1-one-reporter-everywhere) | Customer-managed keys for the data that needs them; no rule ties this to transfers. |

## PECR: device storage and electronic marketing

Privacy and Electronic Communications Regulations 2003 as amended by the DUAA. PECR applies to the SaaS as controller of its own site, app and marketing.

| ID | Item | Repo rule | Rules | Note |
|---|---|---|---|---|
| GDPR-PECR-1 | [C] Storage on or access to a device (cookies, local storage, app storage, SDK identifiers) only when exempt or consented | 🟡 Partial | [18 §1](../18-security-and-privacy.md#1-authentication), [03 §3.3.1](../03-backend-domain-and-ports.md#331-cookies-sessions-and-client-held-state), [03 §3.3.2](../03-backend-domain-and-ports.md#332-the-client-instance-identifier) | The token in browser storage is strictly necessary; no inventory of what clients and SDKs store. |
| GDPR-PECR-2 | [C] Analytics and appearance storage use the DUAA exceptions only for that sole purpose, with clear information and a free, simple way to object | ❌ Gap | [09 §10](../09-frontend.md#10-errors-and-observability-in-the-client), [32 §13](../32-mobile-delivery.md#13-observability-after-release) | Error and crash SDKs store identifiers and may replay sessions; no rule says which exception applies. |
| GDPR-PECR-3 | [C] Consent, where needed, is asked before storage, with refusal as easy as acceptance | ➖ N/A |  | Nothing needs consent as written; this becomes a row when a marketing tag is added. |
| GDPR-PECR-4 | [C] Marketing email and texts to individuals (including sole traders and partnerships) have consent or the soft opt-in, identify the sender and offer a free opt-out | ❌ Gap |  | Corporate addresses are outside reg 22 but a named person's address is still personal data. |
| GDPR-PECR-5 | [C] Service emails (invitations, codes, resets) carry no marketing | 🟡 Partial | [30 §4](../30-sign-in-branding-and-email.md#4-the-email-text-the-custom-message-trigger) | Templates are transactional by design; no rule keeps promotion out. |
| GDPR-PECR-6 | [C+P] Opt-outs are honoured for good and kept per sender | 🟡 Partial | [31 §5.1](../31-multi-tenancy.md#51-the-shapes-r18-and-r19-are-about), [27 §2.1](../27-tool-servers-and-mcp.md#21-reasons-to-keep-an-endpoint-off-the-list) | Suppression list per customer; an opt-out is kept off the tool list. |
---

## DPIA template and LINDDUN prompts

### When a DPIA is required

- **Before** processing that is likely to result in a high risk to people (Art 35(1)). Art 35(3) names three cases, and the ICO's list adds more; the ones a product built to this set meets are **tracking a person's location or behaviour**, **innovative technology** such as a model acting on personal data, and **large-scale processing** of members' data across many customers.
- **Screen every feature that sets `personalData: true`** on a new endpoint ([18](../18-security-and-privacy.md#3-personal-data) § 3), adds a processor, adds a region, or adds a boundary to the threat model ([33](../33-security-review.md#3-threat-model-first) § 3). Record the decision, including "no DPIA needed, because…".
- **As processor, the DPIA is the customer's.** Give them the input: the data flows below, the sub-processor list, the regions, and this checklist's and the ASVS checklist's results. As controller of your own data, write your own.
- **Keep it in the repository**, next to the threat model and the decision records, and change it in the same pull request as the change that alters the processing.

### Template

The headings Art 35(7) requires, in the order of the ICO's DPIA template.

1. **Need for a DPIA.** What the project does, and why it needs a DPIA: which Art 35(3) case or ICO list item it meets.
2. **Description of the processing** (Art 35(7)(a)).
   - *Nature:* how data is collected, stored, used and deleted; the source; who it is shared with; which processors; which transfers. Draw the flow.
   - *Scope:* the categories of data, any special category or criminal-offence data, how much, how often, how long it is kept, how many people, what area.
   - *Context:* the relationship with the people; how much control they have; whether they would expect this; children or other vulnerable people; known concerns; how new the technology is.
   - *Purposes:* what the processing is for, the benefit to the controller, to the people and to others, and the legitimate interest if that is the basis.
3. **Consultation.** The people affected or their representatives, the DPO (Art 35(2)), the processors, security staff. Who was asked, when, and what they said; if no one, why not.
4. **Necessity and proportionality** (Art 35(7)(b)). The lawful basis. Whether the processing achieves the purpose, and whether a less intrusive way would. Minimisation, accuracy, retention. What people are told. How their rights are met. How processors are held to the contract. The safeguards for each transfer.
5. **Risks to people** (Art 35(7)(c)). For each risk: its source, its likelihood (remote, possible, probable), its severity (minimal, significant, severe) and the overall rating (low, medium, high). Risks to people, not to the company.
6. **Measures** (Art 35(7)(d)). For each medium or high risk: the measure, its effect (eliminated, reduced, accepted), the residual risk, and whether it is approved. Each measure becomes a rule, a work item or a decision record ([33](../33-security-review.md#1-the-rules) § 1, rule 4).
7. **Sign-off and outcome.** Who approved the measures and the residual risk. The DPO's advice, and the reason if it was not followed. The consultation responses. **A high residual risk means consulting the Information Commission before the processing starts** (Art 36).
8. **Review.** Who keeps it under review, the next date, and the triggers: a new personal-data endpoint, processor, region or boundary.

### The personal-data flows in this architecture

Model these as the data flow diagram; LINDDUN works on it element by element.

| Flow | From → to | Personal data | Rules |
|---|---|---|---|
| F1 Sign-in | Client → hosted identity service → client storage | Email, name, second-factor secret, tokens | [18](../18-security-and-privacy.md#1-authentication) § 1, [29](../29-hardening-that-costs-nothing.md#2-identity) § 2 |
| F2 Service email | Custom message trigger → mail service → member | Email, name, tenant name, one-time codes | [30](../30-sign-in-branding-and-email.md#4-the-email-text-the-custom-message-trigger) § 4 |
| F3 API call | Client → edge → gateway → dispatcher and privacy gate → use case → store | Whatever the feature holds, keyed by tenant, branch and person id | [18](../18-security-and-privacy.md#3-personal-data) § 3, [06](../06-api-contract-and-codegen.md#62-the-privacy-table-decides-what-leaves-the-building) § 6.2, [08](../08-branches-and-data-scoping.md#1-a-branch-is-not-a-tenant) |
| F4 Files | Client → signed link → object storage | Photos, documents, attachments | [18](../18-security-and-privacy.md#4-input-handling) § 4, [29](../29-hardening-that-costs-nothing.md#3-data) § 3 |
| F5 Mobile sync | Phone offline store ↔ sync endpoints | Location fixes, records, diagnostics snapshot | [10](../10-mobile.md#3-sync) § 3 |
| F6 Telemetry | Clients and API → logs, error tracker, crash reporter | Ids, IP addresses, whatever an exception carries | [17](../17-observability-and-errors.md#5-logging) § 5, [09](../09-frontend.md#10-errors-and-observability-in-the-client) § 10, [32](../32-mobile-delivery.md#13-observability-after-release) § 13 |
| F7 Machine clients | Model ↔ tool server ↔ API; staged payloads | Only what non-personal endpoints return; payload bodies | [27](../27-tool-servers-and-mcp.md#2-the-allowlist-lives-in-the-api-not-in-the-tool-server) § 2, [26](../26-staged-tool-payloads.md#21-the-table) § 2.1 |
| F8 Operators | Platform operator, cross-tenant override, production scripts | Everything, in principle | [31](../31-multi-tenancy.md#53-the-role-split-r20) § 5.3, [18](../18-security-and-privacy.md#7-operating-on-production) § 7 |
| F9 Inbound | Webhooks, inbound mail → API | Senders' details, message content | [18](../18-security-and-privacy.md#4-input-handling) § 4 |
| F10 Lifecycle | Store → export, backup, deletion | All of one customer's data | [31](../31-multi-tenancy.md#8-lifecycle-r26-to-r30) § 8, [29](../29-hardening-that-costs-nothing.md#7-what-is-not-free) § 7 |

### LINDDUN prompts

LINDDUN is the privacy counterpart to the STRIDE pass of [33](../33-security-review.md#3-threat-model-first) § 3. Ask each question of each flow above, and put every "yes" in step 5 of the DPIA.

| Threat | Ask | Start at |
|---|---|---|
| **Linking** | Can records be joined across branches, customers, logs and the error tracker on a person id, an email, an IP address or a device value? Do successive location fixes build a movement profile? Does the client instance identifier outlive one launch? | F3, F5, F6 — [03](../03-backend-domain-and-ports.md#332-the-client-instance-identifier) § 3.3.2, [10](../10-mobile.md#3-sync) § 3 |
| **Identifying** | Does an "anonymous" id in a log or report identify someone once joined with the central user table? Does a file path, file name or public asset name identify a person? Does a location trace reveal a home address? | F4, F6 — [08](../08-branches-and-data-scoping.md#62-static-assets-are-scoped-only-by-their-path) § 6.2, [29](../29-hardening-that-costs-nothing.md#3-data) § 3 |
| **Non-repudiation** | Does a record tie an action or a free-text note to a person who would expect to be able to deny it? Is the audit trail kept longer than its purpose needs? Is it visible to more people than need it? | F3, F8 — [31](../31-multi-tenancy.md#53-the-role-split-r20) § 5.3 |
| **Detecting** | Can a caller tell that a person or record exists from a different error, a 403 against a 404, a timing difference, a push notification on a lock screen, or a realtime channel name? | F1, F3 — [29](../29-hardening-that-costs-nothing.md#2-identity) § 2, [17](../17-observability-and-errors.md#8-the-error-taxonomy-on-the-wire) § 8, [08](../08-branches-and-data-scoping.md#61-push-and-realtime) § 6.1 |
| **Data disclosure** | Does a response carry fields its screen does not show? Does an error report, crash report, diagnostics snapshot, help video or support export carry personal data? Can a machine client reach a personal-data endpoint? Can a staged payload or signed link be read by the wrong caller? Is production data in pre-production? | F3–F8 — [18](../18-security-and-privacy.md#3-personal-data) § 3, [22](../22-pitfalls-checklist.md#4-data-leaking-to-a-lower-privileged-role) § 4, [27](../27-tool-servers-and-mcp.md#2-the-allowlist-lives-in-the-api-not-in-the-tool-server) § 2, [32](../32-mobile-delivery.md#143-data-on-screen) § 14.3 |
| **Unawareness and unintervenability** | Is the member told, at invitation and before the location prompt, what is collected and why? Can the member, not only the administrator, see, correct, export, restrict or object? Can they complain, and does someone answer within 30 days? | F1, F2, F5 — [32](../32-mobile-delivery.md#91-store-review) § 9.1, [30](../30-sign-in-branding-and-email.md#4-the-email-text-the-custom-message-trigger) § 4 |
| **Non-compliance** | Does every store, log, backup and vendor copy have a retention period, and does deletion reach it? Is each purpose in the ROPA with its lawful basis? Is any customer data used for the SaaS's own purposes? Is every sub-processor and transfer listed and covered? | F6, F10 — [31](../31-multi-tenancy.md#lifecycle) R30, [17](../17-observability-and-errors.md#5-logging) § 5 |

---

Sources: [ICO Accountability toolkit](https://ico.org.uk/for-organisations/advice-and-services/audits/data-protection-audit-framework/toolkits/accountability/) (Data Protection Audit Framework, ten sections, 77 control measures; marked under review after the DUAA); [ICO DUAA summary of the changes](https://ico.org.uk/about-the-ico/what-we-do/legislation-we-cover/data-use-and-access-act-2025/the-data-use-and-access-act-2025-duaa-summary-of-the-changes/) (all data protection provisions in force, updated 19 June 2026) and its [data protection](https://ico.org.uk/about-the-ico/what-we-do/legislation-we-cover/data-use-and-access-act-2025/the-data-use-and-access-act-2025-duaa-summary-of-the-changes/data-protection/) and [PECR](https://ico.org.uk/about-the-ico/what-we-do/legislation-we-cover/data-use-and-access-act-2025/the-data-use-and-access-act-2025-duaa-summary-of-the-changes/privacy-and-electronic-communications/) pages; [Bratby Law, DUAA commencement](https://bratby.law/data-use-and-access-act-2025-commencement/); [GOV.UK DUAA commencement plan](https://www.gov.uk/guidance/data-use-and-access-act-2025-plans-for-commencement) (Commencement No. 6, SI 2026/82, in force 5 February 2026, s103 complaints 19 June 2026; Commencement No. 9, SI 2026/1015, Information Commission from 30 September 2026); [GOV.UK, Information Commission succeeds the ICO](https://www.gov.uk/government/news/information-commission-succeeds-the-ico-as-uks-data-protection-regulator); [ICO, a guide to international transfers](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/international-transfers/a-guide-to-international-transfers) (updated 15 January 2026) and [completing a transfer risk assessment](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/international-transfers/completing-a-transfer-risk-assessment); [ICO DPIA guidance](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/accountability-and-governance/data-protection-impact-assessments-dpias/) and its [examples of high-risk processing](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/accountability-and-governance/data-protection-impact-assessments-dpias/examples-of-processing-likely-to-result-in-high-risk/); [LINDDUN threat types](https://linddun.org/threat-types/). Checked 2026-11-02.
