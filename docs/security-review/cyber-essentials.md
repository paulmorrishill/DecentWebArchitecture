# Cyber Essentials checklist

UK Cyber Essentials, **Requirements for IT Infrastructure v3.3** (April 2026, question set "Danzell", in force for assessments started from 27 April 2026), and the **Cyber Essentials Plus Test Specification v3.2** (April 2025), the current one as of November 2026. How to run the review is [33-security-review](../33-security-review.md).

Cyber Essentials certifies an **organisation**, not an application. It covers every device, account and cloud service that touches the organisation's data: the developers' laptops, the git host, the pipeline runners, the cloud account, the app-store consoles and the product's own administrators. The application rules in this set reach only part of that, so expect 🏢 rows.

- **Repo rule.** ✅ a rule here, if followed, meets the item. 🟡 a rule meets part of it. ❌ no rule. ➖ the architecture as written has no such surface. 🏢 an organisational, people or physical control that an architecture cannot cover. **A tick is not a pass.** It says where to look; the reviewer still checks.
- **Tracker.** [cyber-essentials.csv](cyber-essentials.csv) has the same rows with empty result columns.
- **Wording.** Items are paraphrased; the requirement document is the authority. Re-check it before an assessment: IASME changes the question set more often than NCSC changes the requirements.

---

## What a SaaS team gets wrong

- **Cloud services cannot be left out of scope** (new in v3.3). The cloud account, the git and pipeline host, the identity provider, the error tracker, the mail service and the app-store consoles are all cloud services. List them.
- **Multi-factor authentication on every cloud service, for every user, every time.** Since April 2026 a cloud service without it, where the service offers it, is an automatic fail. This set offers a one-time-code second factor in the product ([29](../29-hardening-that-costs-nothing.md) § 2) but no rule requires it for the staff accounts on the services the team uses.
- **Critical and high fixes within 14 days** — CVSS 7 or above, or the vendor gives no score. The release-age wait of [28](../28-supply-chain-and-build-inputs.md) § 3 spends part of that window, so a security fix must not wait the full release age.
- **Self-hosted pipeline runners are devices.** They need updates, malware protection and a firewall like a laptop does, and pinning a runner image by digest freezes its patches.
- **Who does what is split by service model.** For infrastructure the provider runs, the applicant still owns the operating system and above; for a platform service, the application and its configuration; for software as a service, the accounts and the settings. Record the split per service, with the provider's commitment for its part.
- **The application's own code is out of scope**, but v3.3 points to the UK Software Security Code of Practice for it. The ASVS checklist is the stronger test.
- **Backups are recommended, not required.** Do them anyway ([33](../33-security-review.md) § 7).

## Coverage

| | ✅ Covered | 🟡 Partial | ❌ Gap | 🏢 Org | Total |
|---|---|---|---|---|---|
| Scope | 0 | 5 | 1 | 2 | 8 |
| Firewalls | 2 | 3 | 1 | 1 | 7 |
| Secure configuration | 2 | 4 | 0 | 1 | 7 |
| Security update management | 0 | 3 | 2 | 1 | 6 |
| User access control | 1 | 8 | 3 | 0 | 12 |
| Malware protection | 0 | 2 | 1 | 1 | 4 |
| CE Plus | 0 | 2 | 2 | 2 | 6 |
| **All** | 5 | 27 | 10 | 8 | 50 |

## Scope

| ID | Item | Repo rule | Rules | Note |
|---|---|---|---|---|
| CE-SC-1 | Whole organisation or a defined, separately managed sub-set; end-user devices must be in scope | 🏢 Org |  | Scoping decision for the organisation; repo does not define certification scope. |
| CE-SC-2 | Every cloud service holding org data (IaaS/PaaS/SaaS) is in scope and cannot be excluded | 🟡 Partial | [15 §1](../15-infrastructure-and-environments.md#1-layout), [15 §2](../15-infrastructure-and-environments.md#2-environments), [29 §5](../29-hardening-that-costs-nothing.md#5-the-account-and-the-pipeline) | IaC lists deployed resources; no inventory of CI host, identity, stores, monitoring SaaS accounts. |
| CE-SC-3 | Per cloud service, confirm which controls the provider implements, backed by contract or trust-centre statements | ❌ Gap |  | No rule to record shared-responsibility split per provider; an ADR would fit (20 §2). |
| CE-SC-4 | BYOD and home/remote devices accessing org data/services are in scope; ISP routers out, so device firewalls apply | 🏢 Org |  | Workforce device policy; repo is silent on developer laptops beyond 14 §1 local stack. |
| CE-SC-5 | Accounts used by third parties, MSPs and contractors are in scope and must meet the controls | 🟡 Partial | [28 §7](../28-supply-chain-and-build-inputs.md#7-the-pipeline-holds-no-long-lived-credential), [32 §5](../32-mobile-delivery.md#5-credentials-and-signing), [27 §6.4](../27-tool-servers-and-mcp.md#64-the-rules-that-make-the-rest-of-it-safe) | Machine identities federated and scoped; human third-party/contractor accounts not addressed. |
| CE-SC-6 | Commercial web apps in scope; bespoke app components out of scope; follow Software Security Code of Practice | 🟡 Partial | [16 §3](../16-ci-cd.md#3-what-ci-actually-checks), [18](../18-security-and-privacy.md), [28](../28-supply-chain-and-build-inputs.md) | Product code is bespoke so out of CE scope; SDLC rules align but no SSCoP mapping. |
| CE-SC-7 | Asset management (recommended, not a control): authoritative, maintained asset information | 🟡 Partial | [15 §1](../15-infrastructure-and-environments.md#1-layout), [31 R30](../31-multi-tenancy.md) | IaC and deletion lists inventory cloud resources; no device or software-on-device inventory. |
| CE-SC-8 | Backups recommended (not a requirement); turn on automatic backups | 🟡 Partial | [29 §3](../29-hardening-that-costs-nothing.md#3-data), [29 §7](../29-hardening-that-costs-nothing.md#7-what-is-not-free), [31 §4.6](../31-multi-tenancy.md#46-everything-else-that-iterates-customers-r22), [29 §6](../29-hardening-that-costs-nothing.md#6-the-tests-that-keep-them-on) | Deletion protection and per-customer backups; PITR optional (paid); no restore-test rule. |

## Firewalls

| ID | Item | Repo rule | Rules | Note |
|---|---|---|---|---|
| CE-FW-1 | Every in-scope device and cloud service protected by a correctly configured firewall or data-flow policy | 🟡 Partial | [29 §1.1](../29-hardening-that-costs-nothing.md#11-the-flat-rate-plan-is-the-big-one), [29 §1.2](../29-hardening-that-costs-nothing.md#12-the-rest-of-the-edge-all-free) | Edge firewall and origin access control for cloud; laptops and self-hosted runners not addressed. |
| CE-FW-2 | Default firewall/admin passwords changed to strong unique ones, or remote admin disabled | ❌ Gap |  | No rule on cloud root account, device or router admin credentials. |
| CE-FW-3 | Admin interface not internet-reachable unless documented need plus MFA or IP allow list | 🟡 Partial | [15 §5](../15-infrastructure-and-environments.md#5-applying), [29 §5](../29-hardening-that-costs-nothing.md#5-the-account-and-the-pipeline) | Changes go via pipeline; cloud console remains internet-facing with no MFA or allow-list rule. |
| CE-FW-4 | Unauthenticated inbound connections blocked by default | ✅ Covered | [18 §1](../18-security-and-privacy.md#1-authentication), [18 §2](../18-security-and-privacy.md#2-authorization), [29 §1.2](../29-hardening-that-costs-nothing.md#12-the-rest-of-the-edge-all-free), [29 §3](../29-hardening-that-costs-nothing.md#3-data) | Gateway refuses tokenless requests; anonymous endpoints explicit; buckets private behind origin access. |
| CE-FW-5 | Inbound rules approved and documented by an authorised person with the business need | ✅ Covered | [15 §4](../15-infrastructure-and-environments.md#4-reviewing-a-change), [18 §5](../18-security-and-privacy.md#5-least-privilege), [18 §2](../18-security-and-privacy.md#2-authorization), [29 §6](../29-hardening-that-costs-nothing.md#6-the-tests-that-keep-them-on) | Rules are IaC, reviewed as a plan in a PR; guard tests pin scanner rules. |
| CE-FW-6 | Unnecessary firewall rules removed or disabled when no longer needed | 🟡 Partial | [15 §4](../15-infrastructure-and-environments.md#4-reviewing-a-change), [29 §1.1](../29-hardening-that-costs-nothing.md#11-the-flat-rate-plan-is-the-big-one) | Rule budget forces pruning; no periodic rule-review requirement. |
| CE-FW-7 | Software firewall on devices used on untrusted networks (home, public wifi) | 🏢 Org |  | Endpoint configuration for staff devices; not in repo. |

## Secure configuration

| ID | Item | Repo rule | Rules | Note |
|---|---|---|---|---|
| CE-SCFG-1 | Remove or disable unnecessary user accounts (guest, unused admin) | 🟡 Partial | [29 §2](../29-hardening-that-costs-nothing.md#2-identity), [31 R20](../31-multi-tenancy.md) | No open sign-up, role split; nothing on pruning unused cloud/CI/store accounts. |
| CE-SCFG-2 | Change default or guessable account passwords | 🟡 Partial | [18 §1](../18-security-and-privacy.md#1-authentication), [29 §2](../29-hardening-that-costs-nothing.md#2-identity) | App stores no passwords and sets length policy; devices and vendor consoles not covered. |
| CE-SCFG-3 | Remove or disable unnecessary software, utilities and network services | 🟡 Partial | [28 §1](../28-supply-chain-and-build-inputs.md#1-no-new-direct-dependency-without-a-named-decision), [18 §8](../18-security-and-privacy.md#8-dependencies), [18 §9](../18-security-and-privacy.md#9-things-that-quietly-become-security-surfaces), [27 §2](../27-tool-servers-and-mcp.md#2-the-allowlist-lives-in-the-api-not-in-the-tool-server) | Strong for app dependencies and exposed endpoints; nothing for runner or laptop software. |
| CE-SCFG-4 | Disable auto-run that executes files without user authorisation | 🟡 Partial | [28 §2](../28-supply-chain-and-build-inputs.md#2-install-scripts-are-off-everywhere) | Package install scripts off everywhere (analogue); OS auto-run on devices not addressed. |
| CE-SCFG-5 | Users authenticated before access to org data or services | ✅ Covered | [18 §1](../18-security-and-privacy.md#1-authentication), [29 §2](../29-hardening-that-costs-nothing.md#2-identity), [31 R15](../31-multi-tenancy.md) | Managed identity; token verified at edge and gateway before any code. |
| CE-SCFG-6 | Device unlock credential; throttle or lock after 10 attempts; PIN at least 6 characters | 🏢 Org | [10 §6](../10-mobile.md#6-authentication) | Device/MDM policy; mobile app only keeps tokens in platform secure storage. |
| CE-SCFG-7 | Cloud services (IaaS/PaaS/SaaS) configured securely, not left at defaults | ✅ Covered | [29 §1.2](../29-hardening-that-costs-nothing.md#12-the-rest-of-the-edge-all-free), [29 §3](../29-hardening-that-costs-nothing.md#3-data), [29 §5](../29-hardening-that-costs-nothing.md#5-the-account-and-the-pipeline), [29 §6](../29-hardening-that-costs-nothing.md#6-the-tests-that-keep-them-on) | Public access blocked, TLS minimum, encryption, retention; guard tests keep settings on. |

## Security update management

| ID | Item | Repo rule | Rules | Note |
|---|---|---|---|---|
| CE-PATCH-1 | All in-scope software licensed and supported, vendor end-of-support date known | 🟡 Partial | [28 §1.3](../28-supply-chain-and-build-inputs.md#13-what-the-agent-checks-before-it-proposes-a-package) | Package vetting checks maintainers, not support status; no runtime or OS end-of-life tracking. |
| CE-PATCH-2 | Unsupported software removed, or isolated in a sub-set with no internet traffic | ❌ Gap |  | No rule to retire end-of-life runtimes, base images, OS on runners. |
| CE-PATCH-3 | Automatic updates enabled where possible | 🟡 Partial | [29 §5](../29-hardening-that-costs-nothing.md#5-the-account-and-the-pipeline), [28 §3](../28-supply-chain-and-build-inputs.md#3-a-new-version-waits), [28 §5](../28-supply-chain-and-build-inputs.md#5-pipeline-actions-are-pinned-to-a-commit) | Dependency updater weekly, 7-day cooldown; devices and runners not covered. |
| CE-PATCH-4 | Critical/high (CVSS v3 7+) or unrated fixes applied within 14 days of release | 🟡 Partial | [28 §3](../28-supply-chain-and-build-inputs.md#3-a-new-version-waits), [28 §4](../28-supply-chain-and-build-inputs.md#4-advisories-and-where-a-package-came-from), [18 §8](../18-security-and-privacy.md#8-dependencies) | Only newly added advisories fail; existing ones listed with no deadline; cooldown eats 7 of 14 days. |
| CE-PATCH-5 | Patching covers servers, IaaS images, firewalls/routers, and self-hosted runner OS | ❌ Gap | [28 §6](../28-supply-chain-and-build-inputs.md#6-container-and-binary-inputs-are-pinned-by-digest-or-hash), [16 §2.4](../16-ci-cd.md#24-self-hosted-runners), [32 §12](../32-mobile-delivery.md#12-runner-and-device-placement) | Digest-pinned images freeze patches; runner provisioning doc has no patch rule. |
| CE-PATCH-6 | Laptops, tablets and phones (incl. BYOD) kept updated | 🏢 Org |  | Workforce device management; not in repo. |

## User access control

| ID | Item | Repo rule | Rules | Note |
|---|---|---|---|---|
| CE-UAC-1 | A process to create and approve user accounts | 🟡 Partial | [29 §2](../29-hardening-that-costs-nothing.md#2-identity) | Admin-only creation with single-use link for app users; no process for staff cloud/CI accounts. |
| CE-UAC-2 | Unique credentials per user before access to applications or devices | 🟡 Partial | [18 §1](../18-security-and-privacy.md#1-authentication), [28 §7](../28-supply-chain-and-build-inputs.md#7-the-pipeline-holds-no-long-lived-credential), [32 §5](../32-mobile-delivery.md#5-credentials-and-signing) | Per-user app identity and per-run pipeline creds; shared human console accounts not banned. |
| CE-UAC-3 | Remove or disable accounts no longer required (leavers, inactivity period) | ❌ Gap | [31 R29](../31-multi-tenancy.md) | Suspension exists for customers; no leaver or inactivity rule for any account. |
| CE-UAC-4 | MFA wherever available; cloud services must always use MFA (auto-fail since Apr 2026) | 🟡 Partial | [29 §2](../29-hardening-that-costs-nothing.md#2-identity) | TOTP offered in app, not mandated; nothing for cloud console, git/CI host, stores, registries. |
| CE-UAC-5 | Separate accounts used only for administrative activity | 🟡 Partial | [31 R20](../31-multi-tenancy.md), [31 §5.3](../31-multi-tenancy.md#53-the-role-split-r20), [28 §7](../28-supply-chain-and-build-inputs.md#7-the-pipeline-holds-no-long-lived-credential) | Role split for operators and read-only plan role; no separate human admin accounts rule. |
| CE-UAC-6 | Remove special privileges when no longer required (role change) | ❌ Gap | [18 §5](../18-security-and-privacy.md#5-least-privilege) | Least privilege at design time for deployables; no access review or revocation trigger. |
| CE-UAC-7 | Accounts grant only the access the role needs | ✅ Covered | [18 §2](../18-security-and-privacy.md#2-authorization), [18 §5](../18-security-and-privacy.md#5-least-privilege), [31 R12](../31-multi-tenancy.md), [22 §1](../22-pitfalls-checklist.md#1-new-permissions-required) | Role gate, ownership, field filtering; per-deployable roles naming exact resources. |
| CE-UAC-8 | Password brute-force protection: MFA, or throttle (10 per 5 min), or lockout after 10 | 🟡 Partial | [18 §1](../18-security-and-privacy.md#1-authentication), [18 §4](../18-security-and-privacy.md#4-input-handling), [29 §1.2](../29-hardening-that-costs-nothing.md#12-the-rest-of-the-edge-all-free) | Managed identity and rate limits; no explicit throttle or lockout setting required. |
| CE-UAC-9 | Password quality: MFA, or 12+ characters, or 8+ with deny list; no maximum length | 🟡 Partial | [29 §2](../29-hardening-that-costs-nothing.md#2-identity) | Length-based policy stated, but no minimum number, deny list or max-length rule. |
| CE-UAC-10 | No forced expiry or complexity rules; support password managers and unique passwords | 🟡 Partial | [29 §2](../29-hardening-that-costs-nothing.md#2-identity) | Rejects character-class rules; silent on expiry and user guidance. |
| CE-UAC-11 | Passwordless (passkeys/FIDO2) as preferred default | ❌ Gap |  | No passkey or FIDO2 rule anywhere. |
| CE-UAC-12 | Process to change passwords promptly when compromise is known or suspected | 🟡 Partial | [18 §6](../18-security-and-privacy.md#6-secrets), [15 §6](../15-infrastructure-and-environments.md#6-secrets) | Documented rotation for secrets; no user or staff credential compromise procedure. |

## Malware protection

| ID | Item | Repo rule | Rules | Note |
|---|---|---|---|---|
| CE-MAL-1 | Malware protection active on every in-scope laptop/desktop/phone (anti-malware or allow listing) | 🏢 Org |  | Workforce endpoint control; not in repo. |
| CE-MAL-2 | Self-hosted CI/build runners have active malware protection | ❌ Gap | [16 §2.4](../16-ci-cd.md#24-self-hosted-runners), [32 §4.2](../32-mobile-delivery.md#42-runner-hygiene-for-local-builds), [32 §12](../32-mobile-delivery.md#12-runner-and-device-placement) | Runner hygiene cleans secrets and caches; no anti-malware or allow-list requirement. |
| CE-MAL-3 | Application allow listing: only approved, code-signed applications run; current approved list | 🟡 Partial | [28 §1](../28-supply-chain-and-build-inputs.md#1-no-new-direct-dependency-without-a-named-decision), [28 §2](../28-supply-chain-and-build-inputs.md#2-install-scripts-are-off-everywhere), [28 §5](../28-supply-chain-and-build-inputs.md#5-pipeline-actions-are-pinned-to-a-commit), [28 §6](../28-supply-chain-and-build-inputs.md#6-container-and-binary-inputs-are-pinned-by-digest-or-hash) | Supply-chain analogue (approval, SHA/digest/hash pins); not device application allow listing. |
| CE-MAL-4 | Cloud/PaaS malware protection confirmed with provider; uploaded content controlled | 🟡 Partial | [18 §4](../18-security-and-privacy.md#4-input-handling), [29 §3](../29-hardening-that-costs-nothing.md#3-data) | Uploads type- and size-pinned via signed URL; no malware scan of uploads. |

## CE Plus

| ID | Item | Repo rule | Rules | Note |
|---|---|---|---|---|
| CE-PLUS-1 | TC1 external vulnerability scan of all internet-facing IPs/services | 🟡 Partial | [29 §1.2](../29-hardening-that-costs-nothing.md#12-the-rest-of-the-edge-all-free), [29 §1.1](../29-hardening-that-costs-nothing.md#11-the-flat-rate-plan-is-the-big-one) | Hardened edge helps pass; no rule to run external scans. |
| CE-PLUS-2 | TC2 authenticated scan of sampled devices/servers/IaaS: no CVSS 7+ fix older than 14 days | ❌ Gap | [28 §4](../28-supply-chain-and-build-inputs.md#4-advisories-and-where-a-package-came-from) | No device or runner scanning; dependency audit only. |
| CE-PLUS-3 | TC3 malware tests via email attachments and browser downloads on sampled devices | 🏢 Org |  | Workforce endpoint and mail filtering; not in repo. |
| CE-PLUS-4 | TC4 assessor observes MFA on each cloud service for normal and admin users | ❌ Gap | [29 §2](../29-hardening-that-costs-nothing.md#2-identity) | Would fail today: no MFA mandate for staff cloud accounts. |
| CE-PLUS-5 | TC5 account separation: standard user cannot run admin processes without admin credentials | 🏢 Org |  | Endpoint local-admin policy; not in repo. |
| CE-PLUS-6 | Scope and sub-set segregation verified technically; any single failed sub-test fails the assessment | 🟡 Partial | [15 §2](../15-infrastructure-and-environments.md#2-environments), [29 §5](../29-hardening-that-costs-nothing.md#5-the-account-and-the-pipeline) | Separate cloud accounts per environment support segregation evidence. |

---

Sources: [NCSC Cyber Essentials resources](https://www.ncsc.gov.uk/cyberessentials/resources); [Requirements for IT Infrastructure v3.3](https://www.ncsc.gov.uk/files/cyber-essentials-requirements-for-it-infrastructure-v3-3.pdf); [CE Plus Test Specification v3.2](https://www.ncsc.gov.uk/files/cyber-essentials-plus-test-specification-v3-2.pdf); [IASME, April 2026 changes](https://iasme.co.uk/articles/upcoming-changes-to-the-cyber-essentials-scheme-april-2026-update/). Checked 2026-11-02.
