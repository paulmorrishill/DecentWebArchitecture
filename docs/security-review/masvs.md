# MASVS checklist

The 24 controls of the **OWASP Mobile Application Security Verification Standard v2.1.0** (January 2024, the current release), for the mobile app of [10-mobile](../10-mobile.md) and [32-mobile-delivery](../32-mobile-delivery.md). How to run the review is [33-security-review](../33-security-review.md).

MASVS 2 has no levels of its own. The MAS testing profiles attach them to the tests: **L1** for every app, **L2** for an app handling sensitive data — which an app built to this set is, since it holds personal data offline — **P** for privacy, and **R** (resilience against reverse engineering and tampering) only where the threat model asks for it. The RESILIENCE rows are ❌ here because no rule asks for them; whether they are needed is a decision for the threat model ([33](../33-security-review.md) § 3), not a default.

- **Repo rule.** ✅ a rule here, if followed, meets the item. 🟡 a rule meets part of it. ❌ no rule. ➖ the architecture as written has no such surface. 🏢 an organisational, people or physical control that an architecture cannot cover. **A tick is not a pass.** It says where to look; the reviewer still checks.
- **Tracker.** [masvs.csv](masvs.csv) has the same rows with empty result columns. Test each control with the matching MASTG tests.
- **Where the API is the control,** the ASVS row is the one to check: an app-side check that the server does not repeat is not a control.

---

## Coverage

| | ✅ Covered | 🟡 Partial | ❌ Gap | ➖ N/A | Total |
|---|---|---|---|---|---|
| MASVS-STORAGE | 0 | 2 | 0 | 0 | 2 |
| MASVS-CRYPTO | 0 | 0 | 0 | 2 | 2 |
| MASVS-AUTH | 1 | 0 | 1 | 1 | 3 |
| MASVS-NETWORK | 0 | 1 | 1 | 0 | 2 |
| MASVS-PLATFORM | 0 | 0 | 3 | 0 | 3 |
| MASVS-CODE | 2 | 1 | 1 | 0 | 4 |
| MASVS-RESILIENCE | 0 | 0 | 4 | 0 | 4 |
| MASVS-PRIVACY | 0 | 4 | 0 | 0 | 4 |
| **All** | 3 | 8 | 10 | 3 | 24 |

## MASVS-STORAGE

| ID | Control | Repo rule | Rules | Note |
|---|---|---|---|---|
| MASVS-STORAGE-1 | The app securely stores sensitive data. | 🟡 Partial | [10 §6](../10-mobile.md#6-authentication), [18 §1](../18-security-and-privacy.md#1-authentication), [10 §2](../10-mobile.md#2-state) | Tokens in secure storage; embedded offline database holding domain/personal data has no encryption rule. |
| MASVS-STORAGE-2 | The app prevents leakage of sensitive data. | 🟡 Partial | [17 §1](../17-observability-and-errors.md#1-one-reporter-everywhere), [17 §5](../17-observability-and-errors.md#5-logging), [18 §3](../18-security-and-privacy.md#3-personal-data) | Logs and reports carry identifiers only; backups, screenshots, clipboard, keyboard cache not addressed. |

## MASVS-CRYPTO

| ID | Control | Repo rule | Rules | Note |
|---|---|---|---|---|
| MASVS-CRYPTO-1 | The app employs current strong cryptography and uses it according to industry best practices. | ➖ N/A | [10 §6](../10-mobile.md#6-authentication) | App performs no cryptography of its own; relies on platform secure storage and TLS. |
| MASVS-CRYPTO-2 | The app performs key management according to industry best practices. | ➖ N/A | [10 §6](../10-mobile.md#6-authentication) | No app-managed keys; signing keys are build credentials (32 §5). |

## MASVS-AUTH

| ID | Control | Repo rule | Rules | Note |
|---|---|---|---|---|
| MASVS-AUTH-1 | The app uses secure authentication and authorization protocols and follows the relevant best practices. | ✅ Covered | [10 §6](../10-mobile.md#6-authentication), [18 §1](../18-security-and-privacy.md#1-authentication), [29 §2](../29-hardening-that-costs-nothing.md#2-identity) | Direct IdP auth with PKCE; all authorization server-side. |
| MASVS-AUTH-2 | The app performs local authentication securely according to the platform best practices. | ➖ N/A |  | No local or biometric unlock in the architecture. |
| MASVS-AUTH-3 | The app secures sensitive operations with additional authentication. | ❌ Gap |  | No step-up or re-authentication rule for sensitive operations. |

## MASVS-NETWORK

| ID | Control | Repo rule | Rules | Note |
|---|---|---|---|---|
| MASVS-NETWORK-1 | The app secures all network traffic according to the current best practices. | 🟡 Partial | [29 §1.2](../29-hardening-that-costs-nothing.md#12-the-rest-of-the-edge-all-free) | Edge enforces TLS minimum and HTTPS; no rule banning cleartext in app config (ATS/network security config). |
| MASVS-NETWORK-2 | The app performs identity pinning for all remote endpoints under the developer's control. | ❌ Gap |  | No pinning rule. |

## MASVS-PLATFORM

| ID | Control | Repo rule | Rules | Note |
|---|---|---|---|---|
| MASVS-PLATFORM-1 | The app uses IPC mechanisms securely. | ❌ Gap |  | Deep links, intents, URL schemes not addressed. |
| MASVS-PLATFORM-2 | The app uses WebViews securely. | ❌ Gap |  | No WebView rule; sign-in browser type not specified. |
| MASVS-PLATFORM-3 | The app uses the user interface securely. | ❌ Gap |  | No masking or app-switcher screenshot protection for sensitive screens. |

## MASVS-CODE

| ID | Control | Repo rule | Rules | Note |
|---|---|---|---|---|
| MASVS-CODE-1 | The app requires an up-to-date platform version. | ❌ Gap |  | No minimum OS version rule. |
| MASVS-CODE-2 | The app has a mechanism for enforcing app updates. | ✅ Covered | [10 §6](../10-mobile.md#6-authentication), [32 §13](../32-mobile-delivery.md#13-observability-after-release) | Startup version check against a minimum supported version. |
| MASVS-CODE-3 | The app only uses software components without known vulnerabilities. | ✅ Covered | [28 §1](../28-supply-chain-and-build-inputs.md#1-no-new-direct-dependency-without-a-named-decision), [28 §4](../28-supply-chain-and-build-inputs.md#4-advisories-and-where-a-package-came-from), [32 §2.5](../32-mobile-delivery.md#25-adding-a-native-dependency) | Advisory gate and dependency approval cover mobile manifests. |
| MASVS-CODE-4 | The app validates and sanitizes all untrusted inputs. | 🟡 Partial | [10 §4](../10-mobile.md#4-downloads-must-check-their-status), [03 §3.3.1](../03-backend-domain-and-ports.md#331-cookies-sessions-and-client-held-state), [04 §3.1](../04-use-cases.md#31-validation-lives-here) | Downloads checked; server validates; no rule on deep-link or push payload input. |

## MASVS-RESILIENCE

| ID | Control | Repo rule | Rules | Note |
|---|---|---|---|---|
| MASVS-RESILIENCE-1 | The app validates the integrity of the platform. | ❌ Gap |  | No root/jailbreak detection (optional per MASVS risk profile). |
| MASVS-RESILIENCE-2 | The app implements anti-tampering mechanisms. | ❌ Gap |  | No integrity/attestation rule (optional per risk profile). |
| MASVS-RESILIENCE-3 | The app implements anti-static analysis mechanisms. | ❌ Gap |  | No obfuscation rule (optional per risk profile). |
| MASVS-RESILIENCE-4 | The app implements anti-dynamic analysis techniques. | ❌ Gap |  | No anti-debug rule (optional per risk profile). |

## MASVS-PRIVACY

| ID | Control | Repo rule | Rules | Note |
|---|---|---|---|---|
| MASVS-PRIVACY-1 | The app minimizes access to sensitive data and resources. | 🟡 Partial | [18 §3](../18-security-and-privacy.md#3-personal-data), [32 §9.1](../32-mobile-delivery.md#91-store-review) | Responses minimised; background location disclosed; no general permission-minimisation rule. |
| MASVS-PRIVACY-2 | The app prevents identification of the user. | 🟡 Partial | [17 §1](../17-observability-and-errors.md#1-one-reporter-everywhere), [17 §5](../17-observability-and-errors.md#5-logging), [32 §9.1](../32-mobile-delivery.md#91-store-review) | Identifiers only in telemetry; no rule on device identifiers or analytics SDKs. |
| MASVS-PRIVACY-3 | The app is transparent about data collection and usage. | 🟡 Partial | [32 §9.1](../32-mobile-delivery.md#91-store-review) | Privacy-policy link and location disclosure; store privacy labels not mentioned. |
| MASVS-PRIVACY-4 | The app offers user control over their data. | 🟡 Partial | [31 R30](../31-multi-tenancy.md), [31 §8](../31-multi-tenancy.md#8-lifecycle-r26-to-r30) | Deletion list per customer; no in-app per-user account deletion rule. |

---

Source: [OWASP MASVS v2.1.0](https://github.com/OWASP/masvs/releases), [mas.owasp.org](https://mas.owasp.org/). Control statements are OWASP's, CC BY-SA 4.0. Checked 2026-11-02.
