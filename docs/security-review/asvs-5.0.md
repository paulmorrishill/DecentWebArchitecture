# ASVS 5.0 checklist

Every requirement of the OWASP Application Security Verification Standard 5.0.0, with the level it first applies at and whether a rule in this set covers it. How to run the review is [33-security-review](../33-security-review.md).

- **Level.** A requirement marked L1 applies at L1, L2 and L3. One marked L2 applies at L2 and L3. One marked L3 applies at L3 only. Pick the level first ([33](../33-security-review.md) § 2).
- **Repo rule.** ✅ a rule here, if followed, meets the requirement. 🟡 a rule meets part of it. ❌ no rule. ➖ the architecture as written has no such surface. **A tick is not a pass.** It says where to look; the reviewer still checks the code.
- **Tracker.** [asvs-5.0.csv](asvs-5.0.csv) has the same rows with empty result, evidence, owner and date columns. Copy it; do not edit this page to record a result.

Source: [OWASP ASVS 5.0.0](https://github.com/OWASP/ASVS/releases/tag/v5.0.0_release), May 2025. The requirement text is OWASP's, licensed [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); the leading "Verify that" is dropped. This page and the CSV are under the same licence.

---

## Coverage by level

| Level | ✅ Covered | 🟡 Partial | ❌ Gap | ➖ N/A | Total |
|---|---|---|---|---|---|
| L1 | 25 | 32 | 12 | 1 | 70 |
| L2 | 54 | 114 | 61 | 24 | 253 |
| L3 | 58 | 143 | 110 | 34 | 345 |

## Coverage by chapter

| Chapter | ✅ | 🟡 | ❌ | ➖ | Total |
|---|---|---|---|---|---|
| [V1 Encoding and Sanitization](#v1-encoding-and-sanitization) | 2 | 10 | 10 | 8 | 30 |
| [V2 Validation and Business Logic](#v2-validation-and-business-logic) | 4 | 6 | 3 | 0 | 13 |
| [V3 Web Frontend Security](#v3-web-frontend-security) | 6 | 12 | 13 | 0 | 31 |
| [V4 API and Web Service](#v4-api-and-web-service) | 1 | 6 | 7 | 2 | 16 |
| [V5 File Handling](#v5-file-handling) | 2 | 4 | 7 | 0 | 13 |
| [V6 Authentication](#v6-authentication) | 3 | 25 | 12 | 7 | 47 |
| [V7 Session Management](#v7-session-management) | 3 | 7 | 9 | 0 | 19 |
| [V8 Authorization](#v8-authorization) | 6 | 4 | 3 | 0 | 13 |
| [V9 Self-contained Tokens](#v9-self-contained-tokens) | 4 | 3 | 0 | 0 | 7 |
| [V10 OAuth and OIDC](#v10-oauth-and-oidc) | 10 | 12 | 11 | 3 | 36 |
| [V11 Cryptography](#v11-cryptography) | 1 | 11 | 11 | 1 | 24 |
| [V12 Secure Communication](#v12-secure-communication) | 2 | 2 | 7 | 1 | 12 |
| [V13 Configuration](#v13-configuration) | 3 | 10 | 8 | 0 | 21 |
| [V14 Data Protection](#v14-data-protection) | 2 | 10 | 1 | 0 | 13 |
| [V15 Secure Coding and Architecture](#v15-secure-coding-and-architecture) | 3 | 14 | 4 | 0 | 21 |
| [V16 Security Logging and Error Handling](#v16-security-logging-and-error-handling) | 6 | 7 | 4 | 0 | 17 |
| [V17 WebRTC](#v17-webrtc) | 0 | 0 | 0 | 12 | 12 |

---

## V1 Encoding and Sanitization

### V1.1 Encoding and Sanitization Architecture

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V1.1.1 | L2 | Input is decoded or unescaped into a canonical form only once, it is only decoded when encoded data in that form is expected, and that this is done before processing the input further, for example it is not performed after input validation or sanitization. | 🟡 Partial | [25 §3](../25-typed-values-and-serialization.md#3-the-conversion-happens-at-three-boundaries-and-only-there) | Conversion only at three named boundaries; no explicit decode-once/canonicalise rule. |
| V1.1.2 | L2 | The application performs output encoding and escaping either as a final step before being used by the interpreter for which it is intended or by the interpreter itself. | 🟡 Partial | [30 §4.1](../30-sign-in-branding-and-email.md#41-the-handler), [27 §6.3](../27-tool-servers-and-mcp.md#63-the-consent-page) | Escaping at interpolation for email HTML and consent page only; no general rule. |

### V1.2 Injection Prevention

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V1.2.1 | L1 | Output encoding for an HTTP response, HTML document, or XML document is relevant for the context required, such as encoding the relevant characters for HTML elements, HTML attributes, HTML comments, CSS, or HTTP header fields, to avoid changing the message or document structure. | 🟡 Partial | [30 §4.1](../30-sign-in-branding-and-email.md#41-the-handler), [27 §6.3](../27-tool-servers-and-mcp.md#63-the-consent-page) | HTML escaping for server-rendered email and consent page; no context-aware encoding rule elsewhere. |
| V1.2.2 | L1 | When dynamically building URLs, untrusted data is encoded according to its context (e.g., URL encoding or base64url encoding for query or path parameters). Ensure that only safe URL protocols are permitted (e.g., disallow javascript: or data:). | 🟡 Partial | [06 §1](../06-api-contract-and-codegen.md#1-the-transport), [18 §1](../18-security-and-privacy.md#1-authentication), [18 §3](../18-security-and-privacy.md#3-personal-data) | Fixed RPC paths, data in body, no tokens/PII in URLs; no URL-encoding or safe-protocol rule. |
| V1.2.3 | L1 | Output encoding or escaping is used when dynamically building JavaScript content (including JSON), to avoid changing the message or document structure (to avoid JavaScript and JSON injection). | 🟡 Partial | [25 §4](../25-typed-values-and-serialization.md#4-the-converter-registry) | Single serializer for JSON; nothing on building JavaScript content safely. |
| V1.2.4 | L1 | Data selection or database queries (e.g., SQL, HQL, NoSQL, Cypher) use parameterized queries, ORMs, entity frameworks, or are otherwise protected from SQL Injection and other database injection attacks. This is also relevant when writing stored procedures. | ✅ Covered | [05 §2](../05-repositories-and-persistence.md#2-rules), [18 §4](../18-security-and-privacy.md#4-input-handling) | Parameterise every query in every store; no caller input concatenated. |
| V1.2.5 | L1 | The application protects against OS command injection and that operating system calls use parameterized OS queries or use contextual command line output encoding. | ❌ Gap |  | No rule on OS command execution or argument handling. |
| V1.2.6 | L2 | The application protects against LDAP injection vulnerabilities, or that specific security controls to prevent LDAP injection have been implemented. | ➖ N/A |  | No LDAP; authentication is a managed identity service (18 §1). |
| V1.2.7 | L2 | The application is protected against XPath injection attacks by using query parameterization or precompiled queries. | ➖ N/A |  | No XML/XPath; API is JSON RPC (06 §1). |
| V1.2.8 | L2 | LaTeX processors are configured securely (such as not using the "--shell-escape" flag) and an allowlist of commands is used to prevent LaTeX injection attacks. | ➖ N/A |  | No LaTeX processing in the architecture. |
| V1.2.9 | L2 | The application escapes special characters in regular expressions (typically using a backslash) to prevent them from being misinterpreted as metacharacters. | ❌ Gap |  | No rule on escaping user input in regular expressions. |
| V1.2.10 | L3 | The application is protected against CSV and Formula Injection. The application must follow the escaping rules defined in RFC 4180 sections 2.6 and 2.7 when exporting CSV content. Additionally, when exporting to CSV or other spreadsheet formats (such as XLS, XLSX, or ODF), special characters (including '=', '+', '-', '@', '\t' (tab), and '\0' (null character)) must be escaped with a single quote if they appear as the first character in a field value. | ❌ Gap |  | No CSV/formula-injection rule for exports. |

### V1.3 Sanitization

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V1.3.1 | L1 | All untrusted HTML input from WYSIWYG editors or similar is sanitized using a well-known and secure HTML sanitization library or framework feature. | ❌ Gap |  | No HTML sanitization rule for rich-text input. |
| V1.3.2 | L1 | The application avoids the use of eval() or other dynamic code execution features such as Spring Expression Language (SpEL). Where there is no alternative, any user input being included must be sanitized before being executed. | ❌ Gap |  | No ban on eval or dynamic code execution. |
| V1.3.3 | L2 | Data being passed to a potentially dangerous context is sanitized beforehand to enforce safety measures, such as only allowing characters which are safe for this context and trimming input which is too long. | 🟡 Partial | [04 §3.1](../04-use-cases.md#31-validation-lives-here) | Use case trims, range-checks and rejects; no context-specific sanitization rule. |
| V1.3.4 | L2 | User-supplied Scalable Vector Graphics (SVG) scriptable content is validated or sanitized to contain only tags and attributes (such as draw graphics) that are safe for the application, e.g., do not contain scripts and foreignObject. | ❌ Gap |  | No rule for user-supplied SVG (brand logos/images included). |
| V1.3.5 | L2 | The application sanitizes or disables user-supplied scriptable or expression template language content, such as Markdown, CSS or XSL stylesheets, BBCode, or similar. | ❌ Gap |  | No rule for Markdown/CSS/template content from users. |
| V1.3.6 | L2 | The application protects against Server-side Request Forgery (SSRF) attacks, by validating untrusted data against an allowlist of protocols, domains, paths and ports and sanitizing potentially dangerous characters before using the data to call another service. | ❌ Gap |  | No SSRF allowlist rule for server-side outbound requests. |
| V1.3.7 | L2 | The application protects against template injection attacks by not allowing templates to be built based on untrusted input. Where there is no alternative, any untrusted input being included dynamically during template creation must be sanitized or strictly validated. | 🟡 Partial | [30 §4.1](../30-sign-in-branding-and-email.md#41-the-handler) | Email templates are fixed code with escaped values; no general template-injection rule. |
| V1.3.8 | L2 | The application appropriately sanitizes untrusted input before use in Java Naming and Directory Interface (JNDI) queries and that JNDI is configured securely to prevent JNDI injection attacks. | ➖ N/A |  | No Java/JNDI; references are TypeScript and C#. |
| V1.3.9 | L2 | The application sanitizes content before it is sent to memcache to prevent injection attacks. | ➖ N/A |  | No memcache in the architecture. |
| V1.3.10 | L2 | Format strings which might resolve in an unexpected or malicious way when used are sanitized before being processed. | ❌ Gap |  | No rule on untrusted format strings. |
| V1.3.11 | L2 | The application sanitizes user input before passing to mail systems to protect against SMTP or IMAP injection. | 🟡 Partial | [29 §4](../29-hardening-that-costs-nothing.md#4-email), [30 §4.1](../30-sign-in-branding-and-email.md#41-the-handler) | Managed mail service, restricted sender, escaped values; no explicit header-injection rule. |
| V1.3.12 | L3 | Regular expressions are free from elements causing exponential backtracking, and ensure untrusted input is sanitized to mitigate ReDoS or Runaway Regex attacks. | ❌ Gap |  | No ReDoS rule. |

### V1.4 Memory, String, and Unmanaged Code

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V1.4.1 | L2 | The application uses memory-safe string, safer memory copy and pointer arithmetic to detect or prevent stack, buffer, or heap overflows. | ➖ N/A |  | Memory-safe managed languages (TypeScript, C#). |
| V1.4.2 | L2 | Sign, range, and input validation techniques are used to prevent integer overflows. | 🟡 Partial | [25 §2](../25-typed-values-and-serialization.md#2-the-wire-format-is-a-specification-not-an-implementation-detail) | Money as integer minor units, large integers as strings; no general overflow/range rule. |
| V1.4.3 | L2 | Dynamically allocated memory and resources are released, and that references or pointers to freed memory are removed or set to null to prevent dangling pointers and use-after-free vulnerabilities. | ➖ N/A |  | Garbage-collected runtimes; no manual memory management. |

### V1.5 Safe Deserialization

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V1.5.1 | L1 | The application configures XML parsers to use a restrictive configuration and that unsafe features such as resolving external entities are disabled to prevent XML eXternal Entity (XXE) attacks. | ➖ N/A |  | No XML parsing; API and wire format are JSON (06 §1, 25 §2). |
| V1.5.2 | L2 | Deserialization of untrusted data enforces safe input handling, such as using an allowlist of object types or restricting client-defined object types, to prevent deserialization attacks. Deserialization mechanisms that are explicitly defined as insecure must not be used with untrusted input. | ✅ Covered | [25 §4](../25-typed-values-and-serialization.md#4-the-converter-registry), [06 §1](../06-api-contract-and-codegen.md#1-the-transport), [17 §8](../17-observability-and-errors.md#8-the-error-taxonomy-on-the-wire) | JSON into declared contract types via typed converters; undeclared shape refused with 400. |
| V1.5.3 | L3 | Different parsers used in the application for the same data type (e.g., JSON parsers, XML parsers, URL parsers), perform parsing in a consistent way and use the same character encoding mechanism to avoid issues such as JSON Interoperability vulnerabilities or different URI or file parsing behavior being exploited in Remote File Inclusion (RFI) or Server-side Request Forgery (SSRF) attacks. | 🟡 Partial | [25 §2](../25-typed-values-and-serialization.md#2-the-wire-format-is-a-specification-not-an-implementation-detail), [25 §4](../25-typed-values-and-serialization.md#4-the-converter-registry) | One serializer and one wire representation per type; parser consistency not stated generally. |

## V2 Validation and Business Logic

### V2.1 Validation and Business Logic Documentation

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V2.1.1 | L1 | The application's documentation defines input validation rules for how to check the validity of data items against an expected structure. This could be common data formats such as credit card numbers, email addresses, telephone numbers, or it could be an internal data format. | 🟡 Partial | [25 §2](../25-typed-values-and-serialization.md#2-the-wire-format-is-a-specification-not-an-implementation-detail), [06 §4](../06-api-contract-and-codegen.md#4-contract-rules-the-extractor-enforces) | Wire-format table and typed contracts define formats; no per-field validation rule documentation. |
| V2.1.2 | L2 | The application's documentation defines how to validate the logical and contextual consistency of combined data items, such as checking that suburb and ZIP code match. | ❌ Gap |  | No requirement to document cross-field consistency rules. |
| V2.1.3 | L2 | Expectations for business logic limits and validations are documented, including both per-user and globally across the application. | 🟡 Partial | [26 §5](../26-staged-tool-payloads.md#5-cost-ceilings) | Cost ceilings documented for anonymous staging only; no general per-user/global limit documentation. |

### V2.2 Input Validation

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V2.2.1 | L1 | Input is validated to enforce business or functional expectations for that input. This should either use positive validation against an allow list of values, patterns, and ranges, or be based on comparing the input to an expected structure and logical limits according to predefined rules. For L1, this can focus on input which is used to make specific business or security decisions. For L2 and up, this should apply to all input. | ✅ Covered | [04 §3.1](../04-use-cases.md#31-validation-lives-here), [17 §8](../17-observability-and-errors.md#8-the-error-taxonomy-on-the-wire) | Trim, parse, range-check, reject in use case; undeclared shapes refused at transport. |
| V2.2.2 | L1 | The application is designed to enforce input validation at a trusted service layer. While client-side validation improves usability and should be encouraged, it must not be relied upon as a security control. | ✅ Covered | [04 §3.1](../04-use-cases.md#31-validation-lives-here), [18 §1](../18-security-and-privacy.md#1-authentication), [03 §3.3.1](../03-backend-domain-and-ports.md#331-cookies-sessions-and-client-held-state) | Validation in server use case; client claims and client-held state never trusted. |
| V2.2.3 | L2 | The application ensures that combinations of related data items are reasonable according to the pre-defined rules. | 🟡 Partial | [04 §3.1](../04-use-cases.md#31-validation-lives-here) | Business-rule check step exists; combination rules not required to be pre-defined. |

### V2.3 Business Logic Security

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V2.3.1 | L1 | The application will only process business logic flows for the same user in the expected sequential step order and without skipping steps. | 🟡 Partial | [04 §3.1](../04-use-cases.md#31-validation-lives-here) | State-machine check before mutate; no explicit multi-step flow ordering rule. |
| V2.3.2 | L2 | Business logic limits are implemented per the application's documentation to avoid business logic flaws being exploited. | 🟡 Partial | [26 §5](../26-staged-tool-payloads.md#5-cost-ceilings) | Limits enforced for staging endpoint; no general business-limit rule. |
| V2.3.3 | L2 | Transactions are being used at the business logic level such that either a business logic operation succeeds in its entirety or it is rolled back to the previous correct state. | ✅ Covered | [05 §3.1](../05-repositories-and-persistence.md#31-document--key-value-stores), [05 §3.2](../05-repositories-and-persistence.md#32-relational-stores), [07 §5.3](../07-composition-and-entrypoints.md#53-the-outbox) | Explicit transactions for cross-table writes; outbox in same transaction; no transactional downgrade. |
| V2.3.4 | L2 | Business logic level locking mechanisms are used to ensure that limited quantity resources (such as theater seats or delivery slots) cannot be double-booked by manipulating the application's logic. | ✅ Covered | [05 §3.1](../05-repositories-and-persistence.md#31-document--key-value-stores), [08 §4.4](../08-branches-and-data-scoping.md#44-writes-are-guarded) | Conditional writes for create-if-absent and optimistic update, with conflict tests. |
| V2.3.5 | L3 | High-value business logic flows require multi-user approval to prevent unauthorized or accidental actions. This could include but is not limited to large monetary transfers, contract approvals, access to classified information, or safety overrides in manufacturing. | ❌ Gap |  | No multi-user approval rule for high-value business actions. |

### V2.4 Anti-automation

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V2.4.1 | L2 | Anti-automation controls are in place to protect against excessive calls to application functions that could lead to data exfiltration, garbage-data creation, quota exhaustion, rate-limit breaches, denial-of-service, or overuse of costly resources. | 🟡 Partial | [18 §4](../18-security-and-privacy.md#4-input-handling), [29 §1.1](../29-hardening-that-costs-nothing.md#11-the-flat-rate-plan-is-the-big-one), [29 §1.2](../29-hardening-that-costs-nothing.md#12-the-rest-of-the-edge-all-free), [26 §5](../26-staged-tool-payloads.md#5-cost-ceilings) | Rate limits on anonymous endpoints, edge per-IP rule, gateway throttling; no per-user authenticated limits. |
| V2.4.2 | L3 | Business logic flows require realistic human timing, preventing excessively rapid transaction submissions. | ❌ Gap |  | No human-timing control. |

## V3 Web Frontend Security

### V3.1 Web Frontend Security Documentation

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V3.1.1 | L3 | Application documentation states the expected security features that browsers using the application must support (such as HTTPS, HTTP Strict Transport Security (HSTS), Content Security Policy (CSP), and other relevant HTTP security mechanisms). It must also define how the application must behave when some of these features are not available (such as warning the user or blocking access). | 🟡 Partial | [29 §1.2](../29-hardening-that-costs-nothing.md#12-the-rest-of-the-edge-all-free) | Lists edge security headers and TLS; no documented browser feature requirements. |

### V3.2 Unintended Content Interpretation

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V3.2.1 | L1 | Security controls are in place to prevent browsers from rendering content or functionality in HTTP responses in an incorrect context (e.g., when an API, a user-uploaded file or other resource is requested directly). Possible controls could include: not serving the content unless HTTP request header fields (such as Sec-Fetch-\*) indicate it is the correct context, using the sandbox directive of the Content-Security-Policy header field or using the attachment disposition type in the Content-Disposition header field. | 🟡 Partial | [29 §1.2](../29-hardening-that-costs-nothing.md#12-the-rest-of-the-edge-all-free), [29 §3](../29-hardening-that-costs-nothing.md#3-data) | nosniff headers, uploads only via signed links; no Content-Disposition/sandbox rule for direct requests. |
| V3.2.2 | L1 | Content intended to be displayed as text, rather than rendered as HTML, is handled using safe rendering functions (such as createTextNode or textContent) to prevent unintended execution of content such as HTML or JavaScript. | ❌ Gap |  | No rule requiring safe text rendering or banning raw HTML injection in clients. |
| V3.2.3 | L3 | The application avoids DOM clobbering when using client-side JavaScript by employing explicit variable declarations, performing strict type checking, avoiding storing global variables on the document object, and implementing namespace isolation. | ❌ Gap |  | No DOM-clobbering rule. |

### V3.3 Cookie Setup

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V3.3.1 | L1 | Cookies have the 'Secure' attribute set, and if the '\__Host-' prefix is not used for the cookie name, the '__Secure-' prefix must be used for the cookie name. | ❌ Gap |  | Tokens travel in headers (18 §1), but no attribute rules for cookies the entrypoint sets. |
| V3.3.2 | L2 | Each cookie's 'SameSite' attribute value is set according to the purpose of the cookie, to limit exposure to user interface redress attacks and browser-based request forgery attacks, commonly known as cross-site request forgery (CSRF). | ❌ Gap |  | No SameSite rule for cookies. |
| V3.3.3 | L2 | Cookies have the '__Host-' prefix for the cookie name unless they are explicitly designed to be shared with other hosts. | ❌ Gap |  | No __Host- prefix rule. |
| V3.3.4 | L2 | If the value of a cookie is not meant to be accessible to client-side scripts (such as a session token), the cookie must have the 'HttpOnly' attribute set and the same value (e. g. session token) must only be transferred to the client via the 'Set-Cookie' header field. | ❌ Gap |  | Session token kept in local storage, readable by scripts; no HttpOnly rule. |
| V3.3.5 | L3 | When the application writes a cookie, the cookie name and value length combined are not over 4096 bytes. Overly large cookies will not be stored by the browser and therefore not sent with requests, preventing the user from using application functionality which relies on that cookie. | ❌ Gap |  | No cookie size rule. |

### V3.4 Browser Security Mechanism Headers

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V3.4.1 | L1 | A Strict-Transport-Security header field is included on all responses to enforce an HTTP Strict Transport Security (HSTS) policy. A maximum age of at least 1 year must be defined, and for L2 and up, the policy must apply to all subdomains as well. | 🟡 Partial | [29 §1.2](../29-hardening-that-costs-nothing.md#12-the-rest-of-the-edge-all-free), [29 §6](../29-hardening-that-costs-nothing.md#6-the-tests-that-keep-them-on) | Managed headers policy adds HSTS with guard test; max-age/includeSubDomains not specified. |
| V3.4.2 | L1 | The Cross-Origin Resource Sharing (CORS) Access-Control-Allow-Origin header field is a fixed value by the application, or if the Origin HTTP request header field value is used, it is validated against an allowlist of trusted origins. When 'Access-Control-Allow-Origin: *' needs to be used, verify that the response does not include any sensitive information. | 🟡 Partial | [07 §7](../07-composition-and-entrypoints.md#7-response-helpers), [12 §2.1](../12-testing-integration-and-guards.md#21-the-guards-worth-having), [18 §9](../18-security-and-privacy.md#9-things-that-quietly-become-security-surfaces) | CORS headers from one config with guard test; no explicit origin allowlist/no-reflection rule. |
| V3.4.3 | L2 | HTTP responses include a Content-Security-Policy response header field which defines directives to ensure the browser only loads and executes trusted content or resources, in order to limit execution of malicious JavaScript. As a minimum, a global policy must be used which includes the directives object-src 'none' and base-uri 'none' and defines either an allowlist or uses nonces or hashes. For an L3 application, a per-response policy with nonces or hashes must be defined. | 🟡 Partial | [27 §6.3](../27-tool-servers-and-mcp.md#63-the-consent-page), [29 §1.2](../29-hardening-that-costs-nothing.md#12-the-rest-of-the-edge-all-free) | CSP required only on MCP consent page; app CSP not mandated, missing on errors. |
| V3.4.4 | L2 | All HTTP responses contain an 'X-Content-Type-Options: nosniff' header field. This instructs browsers not to use content sniffing and MIME type guessing for the given response, and to require the response's Content-Type header field value to match the destination resource. For example, the response to a request for a style is only accepted if the response's Content-Type is 'text/css'. This also enables the use of the Cross-Origin Read Blocking (CORB) functionality by the browser. | ✅ Covered | [29 §1.2](../29-hardening-that-costs-nothing.md#12-the-rest-of-the-edge-all-free), [29 §6](../29-hardening-that-costs-nothing.md#6-the-tests-that-keep-them-on) | Managed security-headers policy turns sniffing off on every response; guard test keeps it. |
| V3.4.5 | L2 | The application sets a referrer policy to prevent leakage of technically sensitive data to third-party services via the 'Referer' HTTP request header field. This can be done using the Referrer-Policy HTTP response header field or via HTML element attributes. Sensitive data could include path and query data in the URL, and for internal non-public applications also the hostname. | ✅ Covered | [29 §1.2](../29-hardening-that-costs-nothing.md#12-the-rest-of-the-edge-all-free), [29 §6](../29-hardening-that-costs-nothing.md#6-the-tests-that-keep-them-on) | Managed security-headers policy sets referrer policy on every response; guard test. |
| V3.4.6 | L2 | The web application uses the frame-ancestors directive of the Content-Security-Policy header field for every HTTP response to ensure that it cannot be embedded by default and that embedding of specific resources is allowed only when necessary. Note that the X-Frame-Options header field, although supported by browsers, is obsolete and may not be relied upon. | 🟡 Partial | [29 §1.2](../29-hardening-that-costs-nothing.md#12-the-rest-of-the-edge-all-free), [27 §6.3](../27-tool-servers-and-mcp.md#63-the-consent-page) | Frame refusal via managed headers; frame-ancestors CSP only on consent page. |
| V3.4.7 | L3 | The Content-Security-Policy header field specifies a location to report violations. | ❌ Gap |  | No CSP reporting endpoint rule. |
| V3.4.8 | L3 | All HTTP responses that initiate a document rendering (such as responses with Content-Type text/html), include the Cross‑Origin‑Opener‑Policy header field with the same-origin directive or the same-origin-allow-popups directive as required. This prevents attacks that abuse shared access to Window objects, such as tabnabbing and frame counting. | ❌ Gap |  | No Cross-Origin-Opener-Policy rule. |

### V3.5 Browser Origin Separation

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V3.5.1 | L1 | That, if the application does not rely on the CORS preflight mechanism to prevent disallowed cross-origin requests to use sensitive functionality, these requests are validated to ensure they originate from the application itself. This may be done by using and validating anti-forgery tokens or requiring extra HTTP header fields that are not CORS-safelisted request-header fields. This is to defend against browser-based request forgery attacks, commonly known as cross-site request forgery (CSRF). | ✅ Covered | [18 §1](../18-security-and-privacy.md#1-authentication), [03 §3.3.1](../03-backend-domain-and-ports.md#331-cookies-sessions-and-client-held-state), [27 §6.3](../27-tool-servers-and-mcp.md#63-the-consent-page) | Credentials only in Authorization header, never cookies; consent POST checks CSRF token. |
| V3.5.2 | L1 | That, if the application relies on the CORS preflight mechanism to prevent disallowed cross-origin use of sensitive functionality, it is not possible to call the functionality with a request which does not trigger a CORS-preflight request. This may require checking the values of the 'Origin' and 'Content-Type' request header fields or using an extra header field that is not a CORS-safelisted header-field. | ✅ Covered | [18 §1](../18-security-and-privacy.md#1-authentication), [06 §1](../06-api-contract-and-codegen.md#1-the-transport) | Header bearer token and JSON POST force preflight for credentialed calls. |
| V3.5.3 | L1 | HTTP requests to sensitive functionality use appropriate HTTP methods such as POST, PUT, PATCH, or DELETE, and not methods defined by the HTTP specification as "safe" such as HEAD, OPTIONS, or GET. Alternatively, strict validation of the Sec-Fetch-* request header fields can be used to ensure that the request did not originate from an inappropriate cross-origin call, a navigation request, or a resource load (such as an image source) where this is not expected. | ✅ Covered | [06 §1](../06-api-contract-and-codegen.md#1-the-transport), [27 §6.3](../27-tool-servers-and-mcp.md#63-the-consent-page) | All API endpoints are RPC over POST; consent GET answers 405. |
| V3.5.4 | L2 | Separate applications are hosted on different hostnames to leverage the restrictions provided by same-origin policy, including how documents or scripts loaded by one origin can interact with resources from another origin and hostname-based restrictions on cookies. | 🟡 Partial | [15 §8](../15-infrastructure-and-environments.md#8-static-content-and-content-delivery) | Each client surface has own distribution and certificate; separate hostnames implied, not stated. |
| V3.5.5 | L2 | Messages received by the postMessage interface are discarded if the origin of the message is not trusted, or if the syntax of the message is invalid. | ❌ Gap |  | No postMessage origin rule. |
| V3.5.6 | L3 | JSONP functionality is not enabled anywhere across the application to avoid Cross-Site Script Inclusion (XSSI) attacks. | ✅ Covered | [06 §1](../06-api-contract-and-codegen.md#1-the-transport) | POST-only JSON RPC transport leaves no JSONP path. |
| V3.5.7 | L3 | Data requiring authorization is not included in script resource responses, like JavaScript files, to prevent Cross-Site Script Inclusion (XSSI) attacks. | 🟡 Partial | [08 §6.2](../08-branches-and-data-scoping.md#62-static-assets-are-scoped-only-by-their-path), [18 §3](../18-security-and-privacy.md#3-personal-data) | Person-level data never in public static assets; not stated for all authorized data. |
| V3.5.8 | L3 | Authenticated resources (such as images, videos, scripts, and other documents) can be loaded or embedded on behalf of the user only when intended. This can be accomplished by strict validation of the Sec-Fetch-* HTTP request header fields to ensure that the request did not originate from an inappropriate cross-origin call, or by setting a restrictive Cross-Origin-Resource-Policy HTTP response header field to instruct the browser to block returned content. | 🟡 Partial | [29 §3](../29-hardening-that-costs-nothing.md#3-data) | Short-lived signed links for files; no CORP/Sec-Fetch embedding controls. |

### V3.6 External Resource Integrity

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V3.6.1 | L3 | Client-side assets, such as JavaScript libraries, CSS, or web fonts, are only hosted externally (e.g., on a Content Delivery Network) if the resource is static and versioned and Subresource Integrity (SRI) is used to validate the integrity of the asset. If this is not possible, there should be a documented security decision to justify this for each resource. | ❌ Gap |  | No SRI or external-asset hosting rule. |

### V3.7 Other Browser Security Considerations

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V3.7.1 | L2 | The application only uses client-side technologies which are still supported and considered secure. Examples of technologies which do not meet this requirement include NSAPI plugins, Flash, Shockwave, ActiveX, Silverlight, NACL, or client-side Java applets. | 🟡 Partial | [18 §8](../18-security-and-privacy.md#8-dependencies), [28 §4](../28-supply-chain-and-build-inputs.md#4-advisories-and-where-a-package-came-from) | Vulnerable dependency reporting; no rule on deprecated client technologies. |
| V3.7.2 | L2 | The application will only automatically redirect the user to a different hostname or domain (which is not controlled by the application) where the destination appears on an allowlist. | 🟡 Partial | [27 §6.2](../27-tool-servers-and-mcp.md#62-the-flow-step-by-step), [27 §6.4](../27-tool-servers-and-mcp.md#64-the-rules-that-make-the-rest-of-it-safe) | Redirect-URI allowlist for MCP OAuth only; no general open-redirect rule. |
| V3.7.3 | L3 | The application shows a notification when the user is being redirected to a URL outside of the application's control, with an option to cancel the navigation. | 🟡 Partial | [27 §6.3](../27-tool-servers-and-mcp.md#63-the-consent-page) | Consent page shows redirect host before code release; no general external-redirect notice. |
| V3.7.4 | L3 | The application's top-level domain (e.g., site.tld) is added to the public preload list for HTTP Strict Transport Security (HSTS). This ensures that the use of TLS for the application is built directly into the main browsers, rather than relying only on the Strict-Transport-Security response header field. | ❌ Gap |  | No HSTS preload rule. |
| V3.7.5 | L3 | The application behaves as documented (such as warning the user or blocking access) if the browser used to access the application does not support the expected security features. | ❌ Gap |  | No behaviour defined for browsers lacking security features. |

## V4 API and Web Service

### V4.1 Generic Web Service Security

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V4.1.1 | L1 | Every HTTP response with a message body contains a Content-Type header field that matches the actual content of the response, including the charset parameter to specify safe character encoding (e.g., UTF-8, ISO-8859-1) according to IANA Media Types, such as "text/", "/+xml" and "/xml". | ❌ Gap |  | No Content-Type/charset rule for responses. |
| V4.1.2 | L2 | Only user-facing endpoints (intended for manual web-browser access) automatically redirect from HTTP to HTTPS, while other services or endpoints do not implement transparent redirects. This is to avoid a situation where a client is erroneously sending unencrypted HTTP requests, but since the requests are being automatically redirected to HTTPS, the leakage of sensitive data goes undiscovered. | 🟡 Partial | [29 §1.2](../29-hardening-that-costs-nothing.md#12-the-rest-of-the-edge-all-free) | HTTP-to-HTTPS redirect at edge applies to all, not restricted to user-facing endpoints. |
| V4.1.3 | L2 | Any HTTP header field used by the application and set by an intermediary layer, such as a load balancer, a web proxy, or a backend-for-frontend service, cannot be overridden by the end-user. Example headers might include X-Real-IP, X-Forwarded-*, or X-User-ID. | ✅ Covered | [07 §2](../07-composition-and-entrypoints.md#2-building-the-request-scope), [27 §6.4](../27-tool-servers-and-mcp.md#64-the-rules-that-make-the-rest-of-it-safe) | Claims only from verified token, never unverified header; trust one proxy hop/signed viewer header. |
| V4.1.4 | L3 | Only HTTP methods that are explicitly supported by the application or its API (including OPTIONS during preflight requests) can be used and that unused methods are blocked. | 🟡 Partial | [06 §1](../06-api-contract-and-codegen.md#1-the-transport), [07 §3](../07-composition-and-entrypoints.md#3-dispatch-and-authorization), [27 §6.3](../27-tool-servers-and-mcp.md#63-the-consent-page) | POST-only RPC, unknown routes 404, consent GET 405; no explicit unused-method blocking. |
| V4.1.5 | L3 | Per-message digital signatures are used to provide additional assurance on top of transport protections for requests or transactions which are highly sensitive or which traverse a number of systems. | 🟡 Partial | [18 §4](../18-security-and-privacy.md#4-input-handling) | Inbound webhook signatures verified before side effects; no per-message signing for sensitive requests. |

### V4.2 HTTP Message Structure Validation

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V4.2.1 | L2 | All application components (including load balancers, firewalls, and application servers) determine boundaries of incoming HTTP messages using the appropriate mechanism for the HTTP version to prevent HTTP request smuggling. In HTTP/1.x, if a Transfer-Encoding header field is present, the Content-Length header must be ignored per RFC 2616. When using HTTP/2 or HTTP/3, if a Content-Length header field is present, the receiver must ensure that it is consistent with the length of the DATA frames. | ❌ Gap |  | No request-smuggling/message-framing rule. |
| V4.2.2 | L3 | When generating HTTP messages, the Content-Length header field does not conflict with the length of the content as determined by the framing of the HTTP protocol, in order to prevent request smuggling attacks. | ❌ Gap |  | No Content-Length consistency rule. |
| V4.2.3 | L3 | The application does not send nor accept HTTP/2 or HTTP/3 messages with connection-specific header fields such as Transfer-Encoding to prevent response splitting and header injection attacks. | ❌ Gap |  | No HTTP/2 connection-header rule. |
| V4.2.4 | L3 | The application only accepts HTTP/2 and HTTP/3 requests where the header fields and values do not contain any CR (\r), LF (\n), or CRLF (\r\n) sequences, to prevent header injection attacks. | ❌ Gap |  | No CR/LF header-value rule. |
| V4.2.5 | L3 | That, if the application (backend or frontend) builds and sends requests, it uses validation, sanitization, or other mechanisms to avoid creating URIs (such as for API calls) or HTTP request header fields (such as Authorization or Cookie), which are too long to be accepted by the receiving component. This could cause a denial of service, such as when sending an overly long request (e.g., a long cookie header field), which results in the server always responding with an error status. | 🟡 Partial | [06 §5](../06-api-contract-and-codegen.md#5-the-generated-client), [06 §5.1](../06-api-contract-and-codegen.md#51-the-transport-core) | Generated client builds fixed paths; no rule for server-side outbound URI/header construction. |

### V4.3 GraphQL

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V4.3.1 | L2 | A query allowlist, depth limiting, amount limiting, or query cost analysis is used to prevent GraphQL or data layer expression Denial of Service (DoS) as a result of expensive, nested queries. | ➖ N/A |  | No GraphQL; API is RPC over POST (06 §1). |
| V4.3.2 | L2 | GraphQL introspection queries are disabled in the production environment unless the GraphQL API is meant to be used by other parties. | ➖ N/A |  | No GraphQL; API is RPC over POST (06 §1). |

### V4.4 WebSocket

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V4.4.1 | L1 | WebSocket over TLS (WSS) is used for all WebSocket connections. | ❌ Gap |  | Push socket API exists (08 §6.1) but no WSS-only rule. |
| V4.4.2 | L2 | That, during the initial HTTP WebSocket handshake, the Origin header field is checked against a list of origins allowed for the application. | ❌ Gap |  | No Origin check on WebSocket handshake. |
| V4.4.3 | L2 | That, if the application's standard session management cannot be used, dedicated tokens are being used for this, which comply with the relevant Session Management security requirements. | 🟡 Partial | [08 §6.1](../08-branches-and-data-scoping.md#61-push-and-realtime) | Handshake uses the token's claim; no dedicated socket-token rules. |
| V4.4.4 | L2 | Dedicated WebSocket session management tokens are initially obtained or validated through the previously authenticated HTTPS session when transitioning an existing HTTPS session to a WebSocket channel. | 🟡 Partial | [08 §6.1](../08-branches-and-data-scoping.md#61-push-and-realtime) | Branch fixed at handshake from verified token; validation of token at handshake not stated. |

## V5 File Handling

### V5.1 File Handling Documentation

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V5.1.1 | L2 | The documentation defines the permitted file types, expected file extensions, and maximum size (including unpacked size) for each upload feature. Additionally, ensure that the documentation specifies how files are made safe for end-users to download and process, such as how the application behaves when a malicious file is detected. | 🟡 Partial | [18 §4](../18-security-and-privacy.md#4-input-handling), [29 §3](../29-hardening-that-costs-nothing.md#3-data) | Type and size pinned per upload link; no documented per-feature types, unpacked size, malicious handling. |

### V5.2 File Upload and Content

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V5.2.1 | L1 | The application will only accept files of a size which it can process without causing a loss of performance or a denial of service attack. | ✅ Covered | [18 §4](../18-security-and-privacy.md#4-input-handling), [29 §3](../29-hardening-that-costs-nothing.md#3-data), [27 §6.4](../27-tool-servers-and-mcp.md#64-the-rules-that-make-the-rest-of-it-safe), [26 §2](../26-staged-tool-payloads.md#2-the-staging-endpoint) | Size limit pinned in signed upload URL and checked first; oversized bodies refused. |
| V5.2.2 | L1 | When the application accepts a file, either on its own or within an archive such as a zip file, it checks if the file extension matches an expected file extension and validates that the contents correspond to the type represented by the extension. This includes, but is not limited to, checking the initial 'magic bytes', performing image re-writing, and using specialized libraries for file content validation. For L1, this can focus just on files which are used to make specific business or security decisions. For L2 and up, this must apply to all files being accepted. | 🟡 Partial | [18 §4](../18-security-and-privacy.md#4-input-handling), [29 §3](../29-hardening-that-costs-nothing.md#3-data) | Declared content type pinned in signature; no extension or content (magic-byte) validation. |
| V5.2.3 | L2 | The application checks compressed files (e.g., zip, gz, docx, odt) against maximum allowed uncompressed size and against maximum number of files before uncompressing the file. | ❌ Gap |  | No archive uncompressed-size or file-count checks. |
| V5.2.4 | L3 | A file size quota and maximum number of files per user are enforced to ensure that a single user cannot fill up the storage with too many files, or excessively large files. | ❌ Gap |  | No per-user file count or storage quota. |
| V5.2.5 | L3 | The application does not allow uploading compressed files containing symlinks unless this is specifically required (in which case it will be necessary to enforce an allowlist of the files that can be symlinked to). | ❌ Gap |  | No symlink-in-archive rule. |
| V5.2.6 | L3 | The application rejects uploaded images with a pixel size larger than the maximum allowed, to prevent pixel flood attacks. | ❌ Gap |  | No image pixel-size limit. |

### V5.3 File Storage

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V5.3.1 | L1 | Files uploaded or generated by untrusted input and stored in a public folder, are not executed as server-side program code when accessed directly with an HTTP request. | ✅ Covered | [05 §2](../05-repositories-and-persistence.md#2-rules), [18 §4](../18-security-and-privacy.md#4-input-handling), [29 §1.2](../29-hardening-that-costs-nothing.md#12-the-rest-of-the-edge-all-free), [29 §3](../29-hardening-that-costs-nothing.md#3-data) | Bytes only in object storage, no public path, reached via signed links; never executed. |
| V5.3.2 | L1 | When the application creates file paths for file operations, instead of user-submitted filenames, it uses internally generated or trusted data, or if user-submitted filenames or file metadata must be used, strict validation and sanitization must be applied. This is to protect against path traversal, local or remote file inclusion (LFI, RFI), and server-side request forgery (SSRF) attacks. | 🟡 Partial | [03 §2.2](../03-backend-domain-and-ports.md#22-ports-that-are-not-repositories), [31 §6.1](../31-multi-tenancy.md#61-what-is-per-tenant-and-is-not-a-row), [29 §5](../29-hardening-that-costs-nothing.md#5-the-account-and-the-pipeline) | Files stored by key under per-tenant prefix; server-generated keys not mandated. |
| V5.3.3 | L3 | Server-side file processing, such as file decompression, ignores user-provided path information to prevent vulnerabilities such as zip slip. | ❌ Gap |  | No zip-slip/path rule for decompression. |

### V5.4 File Download

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V5.4.1 | L2 | The application validates or ignores user-submitted filenames, including in a JSON, JSONP, or URL parameter and specifies a filename in the Content-Disposition header field in the response. | 🟡 Partial | [18 §4](../18-security-and-privacy.md#4-input-handling) | Inbound file names treated as untrusted; no Content-Disposition filename rule. |
| V5.4.2 | L2 | File names served (e.g., in HTTP response header fields or email attachments) are encoded or sanitized (e.g., following RFC 6266) to preserve document structure and prevent injection attacks. | ❌ Gap |  | No rule encoding served file names in headers or attachments. |
| V5.4.3 | L2 | Files obtained from untrusted sources are scanned by antivirus scanners to prevent serving of known malicious content. | ❌ Gap |  | No antivirus scanning of untrusted files. |

## V6 Authentication

### V6.1 Authentication Documentation

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V6.1.1 | L1 | Application documentation defines how controls such as rate limiting, anti-automation, and adaptive response, are used to defend against attacks such as credential stuffing and password brute force. The documentation must make clear how these controls are configured and prevent malicious account lockout. | 🟡 Partial | [18 §4](../18-security-and-privacy.md#4-input-handling), [29 §1.1](../29-hardening-that-costs-nothing.md#11-the-flat-rate-plan-is-the-big-one), [29 §1.2](../29-hardening-that-costs-nothing.md#12-the-rest-of-the-edge-all-free) | Per-address firewall rate rule, gateway throttling, public-endpoint limits; no documented lockout/adaptive response for sign-in. |
| V6.1.2 | L2 | A list of context-specific words is documented in order to prevent their use in passwords. The list could include permutations of organization names, product names, system identifiers, project codenames, department or role names, and similar. | ❌ Gap |  | No context-specific banned-word list documented. |
| V6.1.3 | L2 | That, if the application includes multiple authentication pathways, these are all documented together with the security controls and authentication strength which must be consistently enforced across them. | 🟡 Partial | [18 §1](../18-security-and-privacy.md#1-authentication), [10 §6](../10-mobile.md#6-authentication), [27 §6.1](../27-tool-servers-and-mcp.md#61-why-the-server-runs-its-own-authorization-surface), [27 §6.5](../27-tool-servers-and-mcp.md#65-the-local-transport) | Hosted web, mobile direct, MCP broker, local transport each described; no single list with consistent strength. |

### V6.2 Password Security

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V6.2.1 | L1 | User set passwords are at least 8 characters in length although a minimum of 15 characters is strongly recommended. | 🟡 Partial | [29 §2](../29-hardening-that-costs-nothing.md#2-identity) | Policy by length mandated but no minimum value stated; provider default — not mandated. |
| V6.2.2 | L1 | Users can change their password. | 🟡 Partial | [30 §4](../30-sign-in-branding-and-email.md#4-the-email-text-the-custom-message-trigger) | Forgot-password reset exists; self-service password change provider default — not mandated. |
| V6.2.3 | L1 | Password change functionality requires the user's current and new password. | 🟡 Partial | [18 §1](../18-security-and-privacy.md#1-authentication) | Delegated to identity provider; current-password check provider default — not mandated. |
| V6.2.4 | L1 | Passwords submitted during account registration or password change are checked against an available set of, at least, the top 3000 passwords which match the application's password policy, e.g. minimum length. | ❌ Gap |  | No common-password list check mandated. |
| V6.2.5 | L1 | Passwords of any composition can be used, without rules limiting the type of characters permitted. There must be no requirement for a minimum number of upper or lower case characters, numbers, or special characters. | ✅ Covered | [29 §2](../29-hardening-that-costs-nothing.md#2-identity) | Password policy by length, not by character classes. |
| V6.2.6 | L1 | Password input fields use type=password to mask the entry. Applications may allow the user to temporarily view the entire masked password, or the last typed character of the password. | 🟡 Partial | [18 §1](../18-security-and-privacy.md#1-authentication), [30 §2.4](../30-sign-in-branding-and-email.md#24-when-the-hosted-page-is-not-enough) | Provider hosted page owns the field; masking provider default — not mandated. |
| V6.2.7 | L1 | "paste" functionality, browser password helpers, and external password managers are permitted. | 🟡 Partial | [18 §1](../18-security-and-privacy.md#1-authentication), [30 §2.4](../30-sign-in-branding-and-email.md#24-when-the-hosted-page-is-not-enough) | Hosted page only; paste/password-manager support provider default — not mandated. |
| V6.2.8 | L1 | The application verifies the user's password exactly as received from the user, without any modifications such as truncation or case transformation. | 🟡 Partial | [18 §1](../18-security-and-privacy.md#1-authentication) | Provider verifies password; no truncation/case rule; provider default — not mandated. |
| V6.2.9 | L2 | Passwords of at least 64 characters are permitted. | 🟡 Partial | [29 §2](../29-hardening-that-costs-nothing.md#2-identity) | No maximum length rule; 64-char support provider default — not mandated. |
| V6.2.10 | L2 | A user's password stays valid until it is discovered to be compromised or the user rotates it. The application must not require periodic credential rotation. | 🟡 Partial | [29 §2](../29-hardening-that-costs-nothing.md#2-identity) | No forced rotation configured, but not explicitly forbidden; provider default — not mandated. |
| V6.2.11 | L2 | The documented list of context specific words is used to prevent easy to guess passwords being created. | ❌ Gap |  | No context-word check on password creation. |
| V6.2.12 | L2 | Passwords submitted during account registration or password changes are checked against a set of breached passwords. | ❌ Gap |  | No breached-password check mandated (provider feature not required). |

### V6.3 General Authentication Security

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V6.3.1 | L1 | Controls to prevent attacks such as credential stuffing and password brute force are implemented according to the application's security documentation. | 🟡 Partial | [18 §4](../18-security-and-privacy.md#4-input-handling), [29 §1.1](../29-hardening-that-costs-nothing.md#11-the-flat-rate-plan-is-the-big-one), [29 §1.2](../29-hardening-that-costs-nothing.md#12-the-rest-of-the-edge-all-free) | Edge per-address rate rule and throttling; sign-in brute-force lockout provider default — not mandated. |
| V6.3.2 | L1 | Default user accounts (e.g., "root", "admin", or "sa") are not present in the application or are disabled. | ❌ Gap |  | No rule forbidding seeded/default admin accounts in deployed environments. |
| V6.3.3 | L2 | Either a multi-factor authentication mechanism or a combination of single-factor authentication mechanisms, must be used in order to access the application. For L3, one of the factors must be a hardware-based authentication mechanism which provides compromise and impersonation resistance against phishing attacks while verifying the intent to authenticate by requiring a user-initiated action (such as a button press on a FIDO hardware key or a mobile phone). Relaxing any of the considerations in this requirement requires a fully documented rationale and a comprehensive set of mitigating controls. | 🟡 Partial | [29 §2](../29-hardening-that-costs-nothing.md#2-identity) | TOTP second factor listed as free setting; not required for all users. |
| V6.3.4 | L2 | That, if the application includes multiple authentication pathways, there are no undocumented pathways and that security controls and authentication strength are enforced consistently. | 🟡 Partial | [18 §1](../18-security-and-privacy.md#1-authentication), [18 §9](../18-security-and-privacy.md#9-things-that-quietly-become-security-surfaces), [27 §6.4](../27-tool-servers-and-mcp.md#64-the-rules-that-make-the-rest-of-it-safe) | Gateway enforces token before code; dev auth-bypass shims banned; no pathway inventory. |
| V6.3.5 | L3 | Users are notified of suspicious authentication attempts (successful or unsuccessful). This may include authentication attempts from an unusual location or client, partially successful authentication (only one of multiple factors), an authentication attempt after a long period of inactivity or a successful authentication after several unsuccessful attempts. | ❌ Gap |  | No user notification of suspicious sign-in attempts. |
| V6.3.6 | L3 | Email is not used as either a single-factor or multi-factor authentication mechanism. | ❌ Gap |  | Password reset relies on emailed code (30 §4); no rule against email as factor. |
| V6.3.7 | L3 | Users are notified after updates to authentication details, such as credential resets or modification of the username or email address. | ❌ Gap |  | No notification after credential or email changes. |
| V6.3.8 | L3 | Valid users cannot be deduced from failed authentication challenges, such as by basing on error messages, HTTP response codes, or different response times. Registration and forgot password functionality must also have this protection. | 🟡 Partial | [29 §2](../29-hardening-that-costs-nothing.md#2-identity) | Same answer for activation resend only; hosted sign-in enumeration protection provider default — not mandated. |

### V6.4 Authentication Factor Lifecycle and Recovery

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V6.4.1 | L1 | System generated initial passwords or activation codes are securely randomly generated, follow the existing password policy, and expire after a short period of time or after they are initially used. These initial secrets must not be permitted to become the long term password. | ✅ Covered | [29 §2](../29-hardening-that-costs-nothing.md#2-identity), [30 §4](../30-sign-in-branding-and-email.md#4-the-email-text-the-custom-message-trigger) | Admin-created accounts; single-use, hashed, expiring activation link/temporary password. |
| V6.4.2 | L1 | Password hints or knowledge-based authentication (so-called "secret questions") are not present. | 🟡 Partial | [18 §1](../18-security-and-privacy.md#1-authentication) | No secret questions in design; hosted page behaviour provider default — not mandated. |
| V6.4.3 | L2 | A secure process for resetting a forgotten password is implemented, that does not bypass any enabled multi-factor authentication mechanisms. | 🟡 Partial | [18 §1](../18-security-and-privacy.md#1-authentication), [30 §4](../30-sign-in-branding-and-email.md#4-the-email-text-the-custom-message-trigger) | Provider-run reset with emailed code; MFA-not-bypassed provider default — not mandated. |
| V6.4.4 | L2 | If a multi-factor authentication factor is lost, evidence of identity proofing is performed at the same level as during enrollment. | ❌ Gap |  | No lost-factor identity proofing process. |
| V6.4.5 | L3 | Renewal instructions for authentication mechanisms which expire are sent with enough time to be carried out before the old authentication mechanism expires, configuring automated reminders if necessary. | ❌ Gap |  | No renewal reminders for expiring authenticators. |
| V6.4.6 | L3 | Administrative users can initiate the password reset process for the user, but that this does not allow them to change or choose the user's password. This prevents a situation where they know the user's password. | 🟡 Partial | [03 §2.2](../03-backend-domain-and-ports.md#22-ports-that-are-not-repositories) | IdentityProviderAdmin port does resets; no rule stopping admin choosing the password. |

### V6.5 General Multi-factor authentication requirements

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V6.5.1 | L2 | Lookup secrets, out-of-band authentication requests or codes, and time-based one-time passwords (TOTPs) are only successfully usable once. | 🟡 Partial | [29 §2](../29-hardening-that-costs-nothing.md#2-identity) | Activation link single-use; TOTP/OOB code single use provider default — not mandated. |
| V6.5.2 | L2 | That, when being stored in the application's backend, lookup secrets with less than 112 bits of entropy (19 random alphanumeric characters or 34 random digits) are hashed with an approved password storage hashing algorithm that incorporates a 32-bit random salt. A standard hash function can be used if the secret has 112 bits of entropy or more. | ➖ N/A |  | No lookup secrets (backup codes) in the design. |
| V6.5.3 | L2 | Lookup secrets, out-of-band authentication code, and time-based one-time password seeds, are generated using a Cryptographically Secure Pseudorandom Number Generator (CSPRNG) to avoid predictable values. | 🟡 Partial | [29 §2](../29-hardening-that-costs-nothing.md#2-identity) | TOTP/code generation by provider; CSPRNG provider default — not mandated. |
| V6.5.4 | L2 | Lookup secrets and out-of-band authentication codes have a minimum of 20 bits of entropy (typically 4 random alphanumeric characters or 6 random digits is sufficient). | 🟡 Partial | [29 §2](../29-hardening-that-costs-nothing.md#2-identity) | Provider-generated codes; entropy provider default — not mandated. |
| V6.5.5 | L2 | Out-of-band authentication requests, codes, or tokens, as well as time-based one-time passwords (TOTPs) have a defined lifetime. Out of band requests must have a maximum lifetime of 10 minutes and for TOTP a maximum lifetime of 30 seconds. | 🟡 Partial | [29 §2](../29-hardening-that-costs-nothing.md#2-identity) | Activation link expires; OOB/TOTP lifetimes provider default — not mandated. |
| V6.5.6 | L3 | Any authentication factor (including physical devices) can be revoked in case of theft or other loss. | ❌ Gap |  | No rule to revoke a lost/stolen authentication factor. |
| V6.5.7 | L3 | Biometric authentication mechanisms are only used as secondary factors together with either something you have or something you know. | ➖ N/A |  | No biometric authentication in the architecture. |
| V6.5.8 | L3 | Time-based one-time passwords (TOTPs) are checked based on a time source from a trusted service and not from an untrusted or client provided time. | 🟡 Partial | [29 §2](../29-hardening-that-costs-nothing.md#2-identity) | TOTP verified by provider server-side; trusted time provider default — not mandated. |

### V6.6 Out-of-Band authentication mechanisms

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V6.6.1 | L2 | Authentication mechanisms using the Public Switched Telephone Network (PSTN) to deliver One-time Passwords (OTPs) via phone or SMS are offered only when the phone number has previously been validated, alternate stronger methods (such as Time based One-time Passwords) are also offered, and the service provides information on their security risks to users. For L3 applications, phone and SMS must not be available as options. | 🟡 Partial | [29 §2](../29-hardening-that-costs-nothing.md#2-identity), [29 §7](../29-hardening-that-costs-nothing.md#7-what-is-not-free) | TOTP preferred over SMS (cost reason); SMS not restricted. |
| V6.6.2 | L2 | Out-of-band authentication requests, codes, or tokens are bound to the original authentication request for which they were generated and are not usable for a previous or subsequent one. | 🟡 Partial | [18 §1](../18-security-and-privacy.md#1-authentication) | Provider binds reset/OOB codes to request; provider default — not mandated. |
| V6.6.3 | L2 | A code based out-of-band authentication mechanism is protected against brute force attacks by using rate limiting. Consider also using a code with at least 64 bits of entropy. | 🟡 Partial | [18 §1](../18-security-and-privacy.md#1-authentication) | OOB code brute-force limiting provider default — not mandated. |
| V6.6.4 | L3 | That, where push notifications are used for multi-factor authentication, rate limiting is used to prevent push bombing attacks. Number matching may also mitigate this risk. | ➖ N/A |  | No push-notification MFA used. |

### V6.7 Cryptographic authentication mechanism

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V6.7.1 | L3 | The certificates used to verify cryptographic authentication assertions are stored in a way protects them from modification. | ➖ N/A |  | No certificate/WebAuthn cryptographic authenticators used. |
| V6.7.2 | L3 | The challenge nonce is at least 64 bits in length, and statistically unique or unique over the lifetime of the cryptographic device. | ➖ N/A |  | No cryptographic authenticator challenge-response used. |

### V6.8 Authentication with an Identity Provider

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V6.8.1 | L2 | That, if the application supports multiple identity providers (IdPs), the user's identity cannot be spoofed via another supported identity provider (eg. by using the same user identifier). The standard mitigation would be for the application to register and identify the user using a combination of the IdP ID (serving as a namespace) and the user's ID in the IdP. | ➖ N/A |  | Single identity provider; no multi-IdP federation. |
| V6.8.2 | L2 | The presence and integrity of digital signatures on authentication assertions (for example on JWTs or SAML assertions) are always validated, rejecting any assertions that are unsigned or have invalid signatures. | ✅ Covered | [18 §1](../18-security-and-privacy.md#1-authentication), [29 §2](../29-hardening-that-costs-nothing.md#2-identity) | Gateway verifies token signature, issuer, audience before any code runs. |
| V6.8.3 | L2 | SAML assertions are uniquely processed and used only once within the validity period to prevent replay attacks. | ➖ N/A |  | SAML not used. |
| V6.8.4 | L2 | That, if an application uses a separate Identity Provider (IdP) and expects specific authentication strength, methods, or recentness for specific functions, the application verifies this using the information returned by the IdP. For example, if OIDC is used, this might be achieved by validating ID Token claims such as 'acr', 'amr', and 'auth_time' (if present). If the IdP does not provide this information, the application must have a documented fallback approach that assumes that the minimum strength authentication mechanism was used (for example, single-factor authentication using username and password). | ❌ Gap |  | No acr/amr/auth_time checks on IdP assertions. |

## V7 Session Management

### V7.1 Session Management Documentation

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V7.1.1 | L2 | The user's session inactivity timeout and absolute maximum session lifetime are documented, are appropriate in combination with other controls, and that the documentation includes justification for any deviations from NIST SP 800-63B re-authentication requirements. | 🟡 Partial | [29 §2](../29-hardening-that-costs-nothing.md#2-identity), [27 §6.4](../27-tool-servers-and-mcp.md#64-the-rules-that-make-the-rest-of-it-safe) | Token and refresh lifetimes are set values; no inactivity timeout documented. |
| V7.1.2 | L2 | The documentation defines how many concurrent (parallel) sessions are allowed for one account as well as the intended behaviors and actions to be taken when the maximum number of active sessions is reached. | ❌ Gap |  | Concurrent session policy not documented. |
| V7.1.3 | L2 | All systems that create and manage user sessions as part of a federated identity management ecosystem (such as SSO systems) are documented along with controls to coordinate session lifetimes, termination, and any other conditions that require re-authentication. | 🟡 Partial | [18 §1](../18-security-and-privacy.md#1-authentication), [27 §6.1](../27-tool-servers-and-mcp.md#61-why-the-server-runs-its-own-authorization-surface), [27 §6.4](../27-tool-servers-and-mcp.md#64-the-rules-that-make-the-rest-of-it-safe) | IdP and MCP broker documented with lifetimes; termination coordination not documented. |

### V7.2 Fundamental Session Management Security

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V7.2.1 | L1 | The application performs all session token verification using a trusted, backend service. | ✅ Covered | [18 §1](../18-security-and-privacy.md#1-authentication), [29 §2](../29-hardening-that-costs-nothing.md#2-identity) | Token verified at gateway/entrypoint before any application code. |
| V7.2.2 | L1 | The application uses either self-contained or reference tokens that are dynamically generated for session management, i.e. not using static API secrets and keys. | ✅ Covered | [18 §1](../18-security-and-privacy.md#1-authentication), [27 §6.2](../27-tool-servers-and-mcp.md#62-the-flow-step-by-step) | Provider-issued tokens; broker issues its own per-session tokens; no static keys. |
| V7.2.3 | L1 | If reference tokens are used to represent user sessions, they are unique and generated using a cryptographically secure pseudo-random number generator (CSPRNG) and possess at least 128 bits of entropy. | 🟡 Partial | [27 §6.1](../27-tool-servers-and-mcp.md#61-why-the-server-runs-its-own-authorization-surface), [27 §6.4](../27-tool-servers-and-mcp.md#64-the-rules-that-make-the-rest-of-it-safe) | Broker opaque tokens via SDK, stored hashed; entropy/CSPRNG not stated. |
| V7.2.4 | L1 | The application generates a new session token on user authentication, including re-authentication, and terminates the current session token. | 🟡 Partial | [27 §6.3](../27-tool-servers-and-mcp.md#63-the-consent-page), [18 §1](../18-security-and-privacy.md#1-authentication) | New tokens per sign-in/approval; terminating prior token on re-auth not stated. |

### V7.3 Session Timeout

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V7.3.1 | L2 | There is an inactivity timeout such that re-authentication is enforced according to risk analysis and documented security decisions. | ❌ Gap |  | No inactivity timeout; browser token refreshed on each response (18 §1). |
| V7.3.2 | L2 | There is an absolute maximum session lifetime such that re-authentication is enforced according to risk analysis and documented security decisions. | 🟡 Partial | [29 §2](../29-hardening-that-costs-nothing.md#2-identity), [27 §6.4](../27-tool-servers-and-mcp.md#64-the-rules-that-make-the-rest-of-it-safe) | Refresh-token lifetime and session-row TTL set; no value or rationale mandated. |

### V7.4 Session Termination

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V7.4.1 | L1 | When session termination is triggered (such as logout or expiration), the application disallows any further use of the session. For reference tokens or stateful sessions, this means invalidating the session data at the application backend. Applications using self-contained tokens will need a solution such as maintaining a list of terminated tokens, disallowing tokens produced before a per-user date and time or rotating a per-user signing key. | 🟡 Partial | [27 §6.2](../27-tool-servers-and-mcp.md#62-the-flow-step-by-step) | Broker revoke deletes token hash; no logout rule for web/mobile token invalidation. |
| V7.4.2 | L1 | The application terminates all active sessions when a user account is disabled or deleted (such as an employee leaving the company). | ❌ Gap |  | No rule terminating sessions on account disable/delete. |
| V7.4.3 | L2 | The application gives the option to terminate all other active sessions after a successful change or removal of any authentication factor (including password change via reset or recovery and, if present, an MFA settings update). | ❌ Gap |  | No option to end other sessions after factor change. |
| V7.4.4 | L2 | All pages that require authentication have easy and visible access to logout functionality. | ❌ Gap |  | No visible-logout requirement. |
| V7.4.5 | L2 | Application administrators are able to terminate active sessions for an individual user or for all users. | ❌ Gap |  | No admin capability to terminate user sessions. |

### V7.5 Defenses Against Session Abuse

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V7.5.1 | L2 | The application requires full re-authentication before allowing modifications to sensitive account attributes which may affect authentication such as email address, phone number, MFA configuration, or other information used in account recovery. | ❌ Gap |  | No re-authentication before sensitive account changes. |
| V7.5.2 | L2 | Users are able to view and (having authenticated again with at least one factor) terminate any or all currently active sessions. | ❌ Gap |  | Users cannot view or terminate active sessions. |
| V7.5.3 | L3 | The application requires further authentication with at least one factor or secondary verification before performing highly sensitive transactions or operations. | ❌ Gap |  | No step-up authentication for sensitive transactions. |

### V7.6 Federated Re-authentication

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V7.6.1 | L2 | Session lifetime and termination between Relying Parties (RPs) and Identity Providers (IdPs) behave as documented, requiring re-authentication as necessary such as when the maximum time between IdP authentication events is reached. | 🟡 Partial | [27 §6.2](../27-tool-servers-and-mcp.md#62-the-flow-step-by-step), [27 §6.4](../27-tool-servers-and-mcp.md#64-the-rules-that-make-the-rest-of-it-safe) | Broker refresh tied to upstream refresh lifetime; RP/IdP behaviour otherwise undocumented. |
| V7.6.2 | L2 | Creation of a session requires either the user's consent or an explicit action, preventing the creation of new application sessions without user interaction. | ✅ Covered | [27 §6.3](../27-tool-servers-and-mcp.md#63-the-consent-page), [27 §6.4](../27-tool-servers-and-mcp.md#64-the-rules-that-make-the-rest-of-it-safe) | Explicit consent POST before any broker session; no silent SSO session creation. |

## V8 Authorization

### V8.1 Authorization Documentation

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V8.1.1 | L1 | Authorization documentation defines rules for restricting function-level and data-specific access based on consumer permissions and resource attributes. | ✅ Covered | [18 §2](../18-security-and-privacy.md#2-authorization), [07 §3](../07-composition-and-entrypoints.md#3-dispatch-and-authorization), [08 §4](../08-branches-and-data-scoping.md#4-the-scoping-mechanism), [31 §1](../31-multi-tenancy.md#1-the-rules) | Route-table role gates plus ownership/tenancy scoping rules documented. |
| V8.1.2 | L2 | Authorization documentation defines rules for field-level access restrictions (both read and write) based on consumer permissions and resource attributes. Note that these rules might depend on other attribute values of the relevant data object, such as state or status. | 🟡 Partial | [18 §2](../18-security-and-privacy.md#2-authorization) | Read-side field filtering via per-role response types; write-side field rules absent. |
| V8.1.3 | L3 | The application's documentation defines the environmental and contextual attributes (including but not limited to, time of day, user location, IP address, or device) that are used in the application to make security decisions, including those pertaining to authentication and authorization. | ❌ Gap |  | No environmental/contextual attributes documented. |
| V8.1.4 | L3 | Authentication and authorization documentation defines how environmental and contextual factors are used in decision-making, in addition to function-level, data-specific, and field-level authorization. This should include the attributes evaluated, thresholds for risk, and actions taken (e.g., allow, challenge, deny, step-up authentication). | ❌ Gap |  | No contextual factors in authN/authZ decisions. |

### V8.2 General Authorization Design

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V8.2.1 | L1 | The application ensures that function-level access is restricted to consumers with explicit permissions. | ✅ Covered | [18 §2](../18-security-and-privacy.md#2-authorization), [07 §3](../07-composition-and-entrypoints.md#3-dispatch-and-authorization) | Dispatcher role gate from generated route table; explicit anonymous role; tests both ways. |
| V8.2.2 | L1 | The application ensures that data-specific access is restricted to consumers with explicit permissions to specific data items to mitigate insecure direct object reference (IDOR) and broken object level authorization (BOLA). | ✅ Covered | [18 §2](../18-security-and-privacy.md#2-authorization), [08 §4](../08-branches-and-data-scoping.md#4-the-scoping-mechanism), [08 §8](../08-branches-and-data-scoping.md#8-tests-that-enforce-isolation) | Ownership checks and scoped repository base class with isolation fixtures. |
| V8.2.3 | L2 | The application ensures that field-level access is restricted to consumers with explicit permissions to specific fields to mitigate broken object property level authorization (BOPLA). | 🟡 Partial | [18 §2](../18-security-and-privacy.md#2-authorization), [22 §4](../22-pitfalls-checklist.md#4-data-leaking-to-a-lower-privileged-role) | Response field leakage prevented per role; no mass-assignment/write-field rule. |
| V8.2.4 | L3 | Adaptive security controls based on a consumer's environmental and contextual attributes (such as time of day, location, IP address, or device) are implemented for authentication and authorization decisions, as defined in the application's documentation. These controls must be applied when the consumer tries to start a new session and also during an existing session. | ❌ Gap |  | No adaptive context-based controls. |

### V8.3 Operation Level Authorization

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V8.3.1 | L1 | The application enforces authorization rules at a trusted service layer and doesn't rely on controls that an untrusted consumer could manipulate, such as client-side JavaScript. | ✅ Covered | [18 §1](../18-security-and-privacy.md#1-authentication), [18 §2](../18-security-and-privacy.md#2-authorization), [27 §7](../27-tool-servers-and-mcp.md#7-what-a-tool-server-must-never-do) | All authorization server-side from verified claims; client claims never trusted. |
| V8.3.2 | L3 | Changes to values on which authorization decisions are made are applied immediately. Where changes cannot be applied immediately, (such as when relying on data in self-contained tokens), there must be mitigating controls to alert when a consumer performs an action when they are no longer authorized to do so and revert the change. Note that this alternative would not mitigate information leakage. | 🟡 Partial | [27 §5](../27-tool-servers-and-mcp.md#5-one-active-tenant-per-login) | Tenant membership judged per request; role claims stale until token expiry. |
| V8.3.3 | L3 | Access to an object is based on the originating subject's (e.g. consumer's) permissions, not on the permissions of any intermediary or service acting on their behalf. For example, if a consumer calls a web service using a self-contained token for authentication, and the service then requests data from a different service, the second service will use the consumer's token, rather than a machine-to-machine token from the first service, to make permission decisions. | ✅ Covered | [27 §1](../27-tool-servers-and-mcp.md#1-the-shape), [27 §6.4](../27-tool-servers-and-mcp.md#64-the-rules-that-make-the-rest-of-it-safe), [27 §7](../27-tool-servers-and-mcp.md#7-what-a-tool-server-must-never-do) | Tool server carries user's own token, holds no standing privilege. |

### V8.4 Other Authorization Considerations

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V8.4.1 | L2 | Multi-tenant applications use cross-tenant controls to ensure consumer operations will never affect tenants with which they do not have permissions to interact. | ✅ Covered | [31 §1](../31-multi-tenancy.md#1-the-rules), [08 §5](../08-branches-and-data-scoping.md#5-resolving-the-branch), [12 §2.1](../12-testing-integration-and-guards.md#21-the-guards-worth-having) | Tenant-first keys, non-code barrier, tenant from verified claim, isolation tests. |
| V8.4.2 | L3 | Access to administrative interfaces incorporates multiple layers of security, including continuous consumer identity verification, device security posture assessment, and contextual risk analysis, ensuring that network location or trusted endpoints are not the sole factors for authorization even though they may reduce the likelihood of unauthorized access. | 🟡 Partial | [31 §1](../31-multi-tenancy.md#1-the-rules), [31 §5.3](../31-multi-tenancy.md#53-the-role-split-r20) | Platform operator role separated and audited; no MFA/device-posture layers mandated. |

## V9 Self-contained Tokens

### V9.1 Token source and integrity

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V9.1.1 | L1 | Self-contained tokens are validated using their digital signature or MAC to protect against tampering before accepting the token's contents. | ✅ Covered | [18 §1](../18-security-and-privacy.md#1-authentication), [29 §2](../29-hardening-that-costs-nothing.md#2-identity) | Gateway verifies signature before any code runs. |
| V9.1.2 | L1 | Only algorithms on an allowlist can be used to create and verify self-contained tokens, for a given context. The allowlist must include the permitted algorithms, ideally only either symmetric or asymmetric algorithms, and must not include the 'None' algorithm. If both symmetric and asymmetric must be supported, additional controls will be needed to prevent key confusion. | 🟡 Partial | [29 §2](../29-hardening-that-costs-nothing.md#2-identity) | Gateway authorizer verifies; algorithm allowlist (no 'none') not stated. |
| V9.1.3 | L1 | Key material that is used to validate self-contained tokens is from trusted pre-configured sources for the token issuer, preventing attackers from specifying untrusted sources and keys. For JWTs and other JWS structures, headers such as 'jku', 'x5u', and 'jwk' must be validated against an allowlist of trusted sources. | ✅ Covered | [29 §2](../29-hardening-that-costs-nothing.md#2-identity), [31 §1](../31-multi-tenancy.md#1-the-rules) | Verified against configured issuer and own clients; token from other system refused. |

### V9.2 Token content

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V9.2.1 | L1 | That, if a validity time span is present in the token data, the token and its content are accepted only if the verification time is within this validity time span. For example, for JWTs, the claims 'nbf' and 'exp' must be verified. | ✅ Covered | [18 §1](../18-security-and-privacy.md#1-authentication) | Gateway refuses expired tokens. |
| V9.2.2 | L2 | The service receiving a token validates the token to be the correct type and is meant for the intended purpose before accepting the token's contents. For example, only access tokens can be accepted for authorization decisions and only ID Tokens can be used for proving user authentication. | 🟡 Partial | [18 §1](../18-security-and-privacy.md#1-authentication), [27 §6.2](../27-tool-servers-and-mcp.md#62-the-flow-step-by-step) | Audience checked, but ID vs access token type not distinguished (ID token forwarded). |
| V9.2.3 | L2 | The service only accepts tokens which are intended for use with that service (audience). For JWTs, this can be achieved by validating the 'aud' claim against an allowlist defined in the service. | ✅ Covered | [18 §1](../18-security-and-privacy.md#1-authentication), [29 §2](../29-hardening-that-costs-nothing.md#2-identity), [27 §6.4](../27-tool-servers-and-mcp.md#64-the-rules-that-make-the-rest-of-it-safe) | Wrong-audience tokens refused at gateway and broker bearer check. |
| V9.2.4 | L2 | That, if a token issuer uses the same private key for issuing tokens to different audiences, the issued tokens contain an audience restriction that uniquely identifies the intended audiences. This will prevent a token from being reused with an unintended audience. If the audience identifier is dynamically provisioned, the token issuer must validate these audiences in order to make sure that they do not result in audience impersonation. | 🟡 Partial | [29 §2](../29-hardening-that-costs-nothing.md#2-identity) | Audience validated against own clients; issuer-side audience restriction provider default — not mandated. |

## V10 OAuth and OIDC

### V10.1 Generic OAuth and OIDC Security

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V10.1.1 | L2 | Tokens are only sent to components that strictly need them. For example, when using a backend-for-frontend pattern for browser-based JavaScript applications, access and refresh tokens shall only be accessible for the backend. | 🟡 Partial | [18 §1](../18-security-and-privacy.md#1-authentication), [27 §6.1](../27-tool-servers-and-mcp.md#61-why-the-server-runs-its-own-authorization-surface) | Broker keeps upstream tokens server-side; browser holds tokens in local storage, no BFF. |
| V10.1.2 | L2 | The client only accepts values from the authorization server (such as the authorization code or ID Token) if these values result from an authorization flow that was initiated by the same user agent session and transaction. This requires that client-generated secrets, such as the proof key for code exchange (PKCE) 'code_verifier', 'state' or OIDC 'nonce', are not guessable, are specific to the transaction, and are securely bound to both the client and the user agent session in which the transaction was started. | 🟡 Partial | [29 §2](../29-hardening-that-costs-nothing.md#2-identity), [27 §6.2](../27-tool-servers-and-mcp.md#62-the-flow-step-by-step) | PKCE mandated binding code to flow; state/nonce checks not stated. |

### V10.2 OAuth Client

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V10.2.1 | L2 | That, if the code flow is used, the OAuth client has protection against browser-based request forgery attacks, commonly known as cross-site request forgery (CSRF), which trigger token requests, either by using proof key for code exchange (PKCE) functionality or checking the 'state' parameter that was sent in the authorization request. | ✅ Covered | [29 §2](../29-hardening-that-costs-nothing.md#2-identity), [27 §6.2](../27-tool-servers-and-mcp.md#62-the-flow-step-by-step) | Authorization-code flow with PKCE required. |
| V10.2.2 | L2 | That, if the OAuth client can interact with more than one authorization server, it has a defense against mix-up attacks. For example, it could require that the authorization server return the 'iss' parameter value and validate it in the authorization response and the token response. | ➖ N/A |  | Clients use a single authorization server (one upstream client). |
| V10.2.3 | L3 | The OAuth client only requests the required scopes (or other authorization parameters) in requests to the authorization server. | ❌ Gap |  | No rule on requesting minimal scopes. |

### V10.3 OAuth Resource Server

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V10.3.1 | L2 | The resource server only accepts access tokens that are intended for use with that service (audience). The audience may be included in a structured access token (such as the 'aud' claim in JWT), or it can be checked using the token introspection endpoint. | ✅ Covered | [18 §1](../18-security-and-privacy.md#1-authentication), [29 §2](../29-hardening-that-costs-nothing.md#2-identity), [27 §6.4](../27-tool-servers-and-mcp.md#64-the-rules-that-make-the-rest-of-it-safe) | Audience/resource checked at gateway and broker. |
| V10.3.2 | L2 | The resource server enforces authorization decisions based on claims from the access token that define delegated authorization. If claims such as 'sub', 'scope', and 'authorization_details' are present, they must be part of the decision. | 🟡 Partial | [27 §2](../27-tool-servers-and-mcp.md#2-the-allowlist-lives-in-the-api-not-in-the-tool-server) | Tool-caller allowlist keyed on audience; no scope-based delegated authorization. |
| V10.3.3 | L2 | If an access control decision requires identifying a unique user from an access token (JWT or related token introspection response), the resource server identifies the user from claims that cannot be reassigned to other users. Typically, it means using a combination of 'iss' and 'sub' claims. | 🟡 Partial | [03 §3.3](../03-backend-domain-and-ports.md#33-ambient-request-state-injected-ports-not-a-context-object), [07 §2](../07-composition-and-entrypoints.md#2-building-the-request-scope) | Caller identity from verified claims; sub+iss uniqueness not stated. |
| V10.3.4 | L2 | That, if the resource server requires specific authentication strength, methods, or recentness, it verifies that the presented access token satisfies these constraints. For example, if present, using the OIDC 'acr', 'amr' and 'auth_time' claims respectively. | ❌ Gap |  | No auth strength/recency check on access tokens. |
| V10.3.5 | L3 | The resource server prevents the use of stolen access tokens or replay of access tokens (from unauthorized parties) by requiring sender-constrained access tokens, either Mutual TLS for OAuth 2 or OAuth 2 Demonstration of Proof of Possession (DPoP). | ❌ Gap |  | No sender-constrained (DPoP/mTLS) tokens. |

### V10.4 OAuth Authorization Server

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V10.4.1 | L1 | The authorization server validates redirect URIs based on a client-specific allowlist of pre-registered URIs using exact string comparison. | ✅ Covered | [27 §6.2](../27-tool-servers-and-mcp.md#62-the-flow-step-by-step), [27 §6.4](../27-tool-servers-and-mcp.md#64-the-rules-that-make-the-rest-of-it-safe) | Redirect-URI allowlist at registration, client read, and authorize; SDK validation. |
| V10.4.2 | L1 | That, if the authorization server returns the authorization code in the authorization response, it can be used only once for a token request. For the second valid request with an authorization code that has already been used to issue an access token, the authorization server must reject a token request and revoke any issued tokens related to the authorization code. | 🟡 Partial | [27 §6.1](../27-tool-servers-and-mcp.md#61-why-the-server-runs-its-own-authorization-surface), [27 §6.2](../27-tool-servers-and-mcp.md#62-the-flow-step-by-step) | Short-lived code via SDK; single use not explicitly required. |
| V10.4.3 | L1 | The authorization code is short-lived. The maximum lifetime can be up to 10 minutes for L1 and L2 applications and up to 1 minute for L3 applications. | ✅ Covered | [27 §6.4](../27-tool-servers-and-mcp.md#64-the-rules-that-make-the-rest-of-it-safe) | Authorization code lifetime 5 minutes. |
| V10.4.4 | L1 | For a given client, the authorization server only allows the usage of grants that this client needs to use. Note that the grants 'token' (Implicit flow) and 'password' (Resource Owner Password Credentials flow) must no longer be used. | 🟡 Partial | [29 §2](../29-hardening-that-costs-nothing.md#2-identity), [27 §6.1](../27-tool-servers-and-mcp.md#61-why-the-server-runs-its-own-authorization-surface) | Code+PKCE flow specified; per-client grant restriction not stated. |
| V10.4.5 | L1 | The authorization server mitigates refresh token replay attacks for public clients, preferably using sender-constrained refresh tokens, i.e., Demonstrating Proof of Possession (DPoP) or Certificate-Bound Access Tokens using mutual TLS (mTLS). For L1 and L2 applications, refresh token rotation may be used. If refresh token rotation is used, the authorization server must invalidate the refresh token after usage, and revoke all refresh tokens for that authorization if an already used and invalidated refresh token is provided. | ❌ Gap |  | No refresh-token rotation or sender-constraining for public clients. |
| V10.4.6 | L2 | That, if the code grant is used, the authorization server mitigates authorization code interception attacks by requiring proof key for code exchange (PKCE). For authorization requests, the authorization server must require a valid 'code_challenge' value and must not accept a 'code_challenge_method' value of 'plain'. For a token request, it must require validation of the 'code_verifier' parameter. | ✅ Covered | [27 §6.1](../27-tool-servers-and-mcp.md#61-why-the-server-runs-its-own-authorization-surface), [29 §2](../29-hardening-that-costs-nothing.md#2-identity) | PKCE required, verified by SDK. |
| V10.4.7 | L2 | If the authorization server supports unauthenticated dynamic client registration, it mitigates the risk of malicious client applications. It must validate client metadata such as any registered URIs, ensure the user's consent, and warn the user before processing an authorization request with an untrusted client application. | ✅ Covered | [27 §6.2](../27-tool-servers-and-mcp.md#62-the-flow-step-by-step), [27 §6.3](../27-tool-servers-and-mcp.md#63-the-consent-page), [27 §6.4](../27-tool-servers-and-mcp.md#64-the-rules-that-make-the-rest-of-it-safe) | Open DCR gated by redirect allowlist and per-authorization consent page. |
| V10.4.8 | L2 | Refresh tokens have an absolute expiration, including if sliding refresh token expiration is applied. | ✅ Covered | [27 §6.4](../27-tool-servers-and-mcp.md#64-the-rules-that-make-the-rest-of-it-safe) | Refresh lifetime equals upstream; session row has TTL. |
| V10.4.9 | L2 | Refresh tokens and reference access tokens can be revoked by an authorized user using the authorization server user interface, to mitigate the risk of malicious clients or stolen tokens. | 🟡 Partial | [27 §6.2](../27-tool-servers-and-mcp.md#62-the-flow-step-by-step) | Revoke endpoint deletes by hash; no user-facing revocation UI. |
| V10.4.10 | L2 | Confidential client is authenticated for client-to-authorized server backchannel requests such as token requests, pushed authorization requests (PAR), and token revocation requests. | 🟡 Partial | [27 §6.1](../27-tool-servers-and-mcp.md#61-why-the-server-runs-its-own-authorization-surface) | Client authentication delegated to SDK; method not specified. |
| V10.4.11 | L2 | The authorization server configuration only assigns the required scopes to the OAuth client. | ❌ Gap |  | No scope assignment rules for OAuth clients. |
| V10.4.12 | L3 | For a given client, the authorization server only allows the 'response_mode' value that this client needs to use. For example, by having the authorization server validate this value against the expected values or by using pushed authorization request (PAR) or JWT-secured Authorization Request (JAR). | ❌ Gap |  | response_mode not restricted per client. |
| V10.4.13 | L3 | Grant type 'code' is always used together with pushed authorization requests (PAR). | ❌ Gap |  | PAR not required. |
| V10.4.14 | L3 | The authorization server issues only sender-constrained (Proof-of-Possession) access tokens, either with certificate-bound access tokens using mutual TLS (mTLS) or DPoP-bound access tokens (Demonstration of Proof of Possession). | ❌ Gap |  | Access tokens are bearer, not sender-constrained. |
| V10.4.15 | L3 | That, for a server-side client (which is not executed on the end-user device), the authorization server ensures that the 'authorization_details' parameter value is from the client backend and that the user has not tampered with it. For example, by requiring the usage of pushed authorization request (PAR) or JWT-secured Authorization Request (JAR). | ➖ N/A |  | Rich authorization requests (authorization_details) not used. |
| V10.4.16 | L3 | The client is confidential and the authorization server requires the use of strong client authentication methods (based on public-key cryptography and resistant to replay attacks), such as mutual TLS ('tls_client_auth', 'self_signed_tls_client_auth') or private key JWT ('private_key_jwt'). | ❌ Gap |  | No strong (private_key_jwt/mTLS) client authentication required. |

### V10.5 OIDC Client

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V10.5.1 | L2 | The client (as the relying party) mitigates ID Token replay attacks. For example, by ensuring that the 'nonce' claim in the ID Token matches the 'nonce' value sent in the authentication request to the OpenID Provider (in OAuth2 refereed to as the authorization request sent to the authorization server). | ❌ Gap |  | No ID token nonce/replay check stated. |
| V10.5.2 | L2 | The client uniquely identifies the user from ID Token claims, usually the 'sub' claim, which cannot be reassigned to other users (for the scope of an identity provider). | 🟡 Partial | [03 §3.3](../03-backend-domain-and-ports.md#33-ambient-request-state-injected-ports-not-a-context-object), [07 §2](../07-composition-and-entrypoints.md#2-building-the-request-scope) | Identity from verified claims; non-reassignable sub not stated. |
| V10.5.3 | L2 | The client rejects attempts by a malicious authorization server to impersonate another authorization server through authorization server metadata. The client must reject authorization server metadata if the issuer URL in the authorization server metadata does not exactly match the pre-configured issuer URL expected by the client. | 🟡 Partial | [29 §2](../29-hardening-that-costs-nothing.md#2-identity) | Single provider and issuer checked at gateway; metadata issuer validation not stated. |
| V10.5.4 | L2 | The client validates that the ID Token is intended to be used for that client (audience) by checking that the 'aud' claim from the token is equal to the 'client_id' value for the client. | ✅ Covered | [27 §6.2](../27-tool-servers-and-mcp.md#62-the-flow-step-by-step), [27 §8](../27-tool-servers-and-mcp.md#8-tests) | Broker checks ID token audience is expected client; tested. |
| V10.5.5 | L2 | That, when using OIDC back-channel logout, the relying party mitigates denial of service through forced logout and cross-JWT confusion in the logout flow. The client must verify that the logout token is correctly typed with a value of 'logout+jwt', contains the 'event' claim with the correct member name, and does not contain a 'nonce' claim. Note that it is also recommended to have a short expiration (e.g., 2 minutes). | ➖ N/A |  | OIDC back-channel logout not used. |

### V10.6 OpenID Provider

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V10.6.1 | L2 | The OpenID Provider only allows values 'code', 'ciba', 'id_token', or 'id_token code' for response mode. Note that 'code' is preferred over 'id_token code' (the OIDC Hybrid flow), and 'token' (any Implicit flow) must not be used. | 🟡 Partial | [29 §2](../29-hardening-that-costs-nothing.md#2-identity) | Clients use code flow; OP response modes provider default — not mandated. |
| V10.6.2 | L2 | The OpenID Provider mitigates denial of service through forced logout. By obtaining explicit confirmation from the end-user or, if present, validating parameters in the logout request (initiated by the relying party), such as the 'id_token_hint'. | ❌ Gap |  | No forced-logout DoS mitigation at OP. |

### V10.7 Consent Management

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V10.7.1 | L2 | The authorization server ensures that the user consents to each authorization request. If the identity of the client cannot be assured, the authorization server must always explicitly prompt the user for consent. | ✅ Covered | [27 §6.3](../27-tool-servers-and-mcp.md#63-the-consent-page), [27 §7](../27-tool-servers-and-mcp.md#7-what-a-tool-server-must-never-do) | Consent on every authorization; no remembered approval. |
| V10.7.2 | L2 | When the authorization server prompts for user consent, it presents sufficient and clear information about what is being consented to. When applicable, this should include the nature of the requested authorizations (typically based on scope, resource server, Rich Authorization Requests (RAR) authorization details), the identity of the authorized application, and the lifetime of these authorizations. | ✅ Covered | [27 §6.3](../27-tool-servers-and-mcp.md#63-the-consent-page) | Shows allowlisted client name, redirect host, account identity and roles. |
| V10.7.3 | L2 | The user can review, modify, and revoke consents which the user has granted through the authorization server. | 🟡 Partial | [27 §6.3](../27-tool-servers-and-mcp.md#63-the-consent-page), [27 §6.2](../27-tool-servers-and-mcp.md#62-the-flow-step-by-step) | No stored consents to review; issued grants not user-reviewable/revocable. |

## V11 Cryptography

### V11.1 Cryptographic Inventory and Documentation

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V11.1.1 | L2 | There is a documented policy for management of cryptographic keys and a cryptographic key lifecycle that follows a key management standard such as NIST SP 800-57. This should include ensuring that keys are not overshared (for example, with more than two entities for shared secrets and more than one entity for private keys). | 🟡 Partial | [18 §6](../18-security-and-privacy.md#6-secrets), [15 §6](../15-infrastructure-and-environments.md#6-secrets), [29 §7](../29-hardening-that-costs-nothing.md#7-what-is-not-free) | Rotation procedure per secret and CMK use; no key lifecycle policy. |
| V11.1.2 | L2 | A cryptographic inventory is performed, maintained, regularly updated, and includes all cryptographic keys, algorithms, and certificates used by the application. It must also document where keys can and cannot be used in the system, and the types of data that can and cannot be protected using the keys. | ❌ Gap |  | No cryptographic inventory. |
| V11.1.3 | L3 | Cryptographic discovery mechanisms are employed to identify all instances of cryptography in the system, including encryption, hashing, and signing operations. | ❌ Gap |  | No cryptographic discovery. |
| V11.1.4 | L3 | A cryptographic inventory is maintained. This must include a documented plan that outlines the migration path to new cryptographic standards, such as post-quantum cryptography, in order to react to future threats. | ❌ Gap |  | No crypto migration/PQC plan. |

### V11.2 Secure Cryptography Implementation

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V11.2.1 | L2 | Industry-validated implementations (including libraries and hardware-accelerated implementations) are used for cryptographic operations. | ✅ Covered | [27 §6.1](../27-tool-servers-and-mcp.md#61-why-the-server-runs-its-own-authorization-surface), [18 §1](../18-security-and-privacy.md#1-authentication), [30 §6](../30-sign-in-branding-and-email.md#6-the-custom-email-sender-and-why-it-is-the-second-choice) | Hand-rolled crypto banned; managed services and provider SDKs used. |
| V11.2.2 | L2 | The application is designed with crypto agility such that random number, authenticated encryption, MAC, or hashing algorithms, key lengths, rounds, ciphers and modes can be reconfigured, upgraded, or swapped at any time, to protect against cryptographic breaks. Similarly, it must also be possible to replace keys and passwords and re-encrypt data. This will allow for seamless upgrades to post-quantum cryptography (PQC), once high-assurance implementations of approved PQC schemes or standards are widely available. | ❌ Gap |  | No crypto-agility requirement. |
| V11.2.3 | L2 | All cryptographic primitives utilize a minimum of 128-bits of security based on the algorithm, key size, and configuration. For example, a 256-bit ECC key provides roughly 128 bits of security where RSA requires a 3072-bit key to achieve 128 bits of security. | ❌ Gap |  | No minimum security-strength rule for primitives. |
| V11.2.4 | L3 | All cryptographic operations are constant-time, with no 'short-circuit' operations in comparisons, calculations, or returns, to avoid leaking information. | ❌ Gap |  | No constant-time operation rule. |
| V11.2.5 | L3 | All cryptographic modules fail securely, and errors are handled in a way that does not enable vulnerabilities, such as Padding Oracle attacks. | ❌ Gap |  | No secure-failure rule for crypto modules. |

### V11.3 Encryption Algorithms

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V11.3.1 | L1 | Insecure block modes (e.g., ECB) and weak padding schemes (e.g., PKCS#1 v1.5) are not used. | 🟡 Partial | [29 §3](../29-hardening-that-costs-nothing.md#3-data), [27 §6.1](../27-tool-servers-and-mcp.md#61-why-the-server-runs-its-own-authorization-surface) | Encryption via managed provider defaults; modes/padding not mandated. |
| V11.3.2 | L1 | Only approved ciphers and modes such as AES with GCM are used. | 🟡 Partial | [29 §3](../29-hardening-that-costs-nothing.md#3-data), [27 §6.1](../27-tool-servers-and-mcp.md#61-why-the-server-runs-its-own-authorization-surface) | Managed encryption at rest; approved ciphers not specified. |
| V11.3.3 | L2 | Encrypted data is protected against unauthorized modification preferably by using an approved authenticated encryption method or by combining an approved encryption method with an approved MAC algorithm. | 🟡 Partial | [18 §6.1](../18-security-and-privacy.md#61-asking-a-human-for-a-credential), [29 §3](../29-hardening-that-costs-nothing.md#3-data) | Agent secret vault uses authenticated header; general rule absent. |
| V11.3.4 | L3 | Nonces, initialization vectors, and other single-use numbers are not used for more than one encryption key and data-element pair. The method of generation must be appropriate for the algorithm being used. | ❌ Gap |  | No nonce/IV uniqueness rule. |
| V11.3.5 | L3 | Any combination of an encryption algorithm and a MAC algorithm is operating in encrypt-then-MAC mode. | ❌ Gap |  | No encrypt-then-MAC rule. |

### V11.4 Hashing and Hash-based Functions

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V11.4.1 | L1 | Only approved hash functions are used for general cryptographic use cases, including digital signatures, HMAC, KDF, and random bit generation. Disallowed hash functions, such as MD5, must not be used for any cryptographic purpose. | 🟡 Partial | [27 §6.2](../27-tool-servers-and-mcp.md#62-the-flow-step-by-step), [28 §6](../28-supply-chain-and-build-inputs.md#6-container-and-binary-inputs-are-pinned-by-digest-or-hash) | SHA-256 for token hashes and build pins; no general approved-hash rule. |
| V11.4.2 | L2 | Passwords are stored using an approved, computationally intensive, key derivation function (also known as a "password hashing function"), with parameter settings configured based on current guidance. The settings should balance security and performance to make brute-force attacks sufficiently challenging for the required level of security. | 🟡 Partial | [18 §1](../18-security-and-privacy.md#1-authentication) | App stores no passwords; provider password storage provider default — not mandated. |
| V11.4.3 | L2 | Hash functions used in digital signatures, as part of data authentication or data integrity are collision resistant and have appropriate bit-lengths. If collision resistance is required, the output length must be at least 256 bits. If only resistance to second pre-image attacks is required, the output length must be at least 128 bits. | 🟡 Partial | [28 §6](../28-supply-chain-and-build-inputs.md#6-container-and-binary-inputs-are-pinned-by-digest-or-hash), [27 §6.4](../27-tool-servers-and-mcp.md#64-the-rules-that-make-the-rest-of-it-safe) | Hash pinning of build inputs; algorithm strength not specified. |
| V11.4.4 | L2 | The application uses approved key derivation functions with key stretching parameters when deriving secret keys from passwords. The parameters in use must balance security and performance to prevent brute-force attacks from compromising the resulting cryptographic key. | ➖ N/A |  | No keys derived from passwords in the design. |

### V11.5 Random Values

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V11.5.1 | L2 | All random numbers and strings which are intended to be non-guessable must be generated using a cryptographically secure pseudo-random number generator (CSPRNG) and have at least 128 bits of entropy. Note that UUIDs do not respect this condition. | 🟡 Partial | [26 §2](../26-staged-tool-payloads.md#2-the-staging-endpoint), [01 §6](../01-principles.md#6-time-identity-and-randomness-are-injected) | Server-made crypto randomUUID for staged ids; no general CSPRNG rule. |
| V11.5.2 | L3 | The random number generation mechanism in use is designed to work securely, even under heavy demand. | ❌ Gap |  | No RNG-under-load requirement. |

### V11.6 Public Key Cryptography

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V11.6.1 | L2 | Only approved cryptographic algorithms and modes of operation are used for key generation and seeding, and digital signature generation and verification. Key generation algorithms must not generate insecure keys vulnerable to known attacks, for example, RSA keys which are vulnerable to Fermat factorization. | 🟡 Partial | [18 §1](../18-security-and-privacy.md#1-authentication), [29 §7](../29-hardening-that-costs-nothing.md#7-what-is-not-free) | Token signing delegated to provider, CMKs for sensitive data; algorithms not mandated. |
| V11.6.2 | L3 | Approved cryptographic algorithms are used for key exchange (such as Diffie-Hellman) with a focus on ensuring that key exchange mechanisms use secure parameters. This will prevent attacks on the key establishment process which could lead to adversary-in-the-middle attacks or cryptographic breaks. | 🟡 Partial | [29 §1.2](../29-hardening-that-costs-nothing.md#12-the-rest-of-the-edge-all-free) | Minimum TLS version at edge; key-exchange algorithms not specified. |

### V11.7 In-Use Data Cryptography

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V11.7.1 | L3 | Full memory encryption is in use that protects sensitive data while it is in use, preventing access by unauthorized users or processes. | ❌ Gap |  | No memory encryption requirement. |
| V11.7.2 | L3 | Data minimization ensures the minimal amount of data is exposed during processing, and ensure that data is encrypted immediately after use or as soon as feasible. | 🟡 Partial | [18 §3](../18-security-and-privacy.md#3-personal-data), [29 §3](../29-hardening-that-costs-nothing.md#3-data) | Data minimisation in responses and records; no encrypt-after-use rule. |

## V12 Secure Communication

### V12.1 General TLS Security Guidance

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V12.1.1 | L1 | Only the latest recommended versions of the TLS protocol are enabled, such as TLS 1.2 and TLS 1.3. The latest version of the TLS protocol must be the preferred option. | 🟡 Partial | [29 §1.2](../29-hardening-that-costs-nothing.md#12-the-rest-of-the-edge-all-free), [15 §8](../15-infrastructure-and-environments.md#8-static-content-and-content-delivery) | Minimum TLS version set at edge; no rule naming TLS 1.2/1.3 or preferring 1.3. |
| V12.1.2 | L2 | Only recommended cipher suites are enabled, with the strongest cipher suites set as preferred. L3 applications must only support cipher suites which provide forward secrecy. | ❌ Gap |  | No rule on cipher suites or forward secrecy. |
| V12.1.3 | L2 | The application validates that mTLS client certificates are trusted before using the certificate identity for authentication or authorization. | ➖ N/A |  | No mTLS client certificates in the architecture; auth is managed-identity tokens. |
| V12.1.4 | L3 | Proper certification revocation, such as Online Certificate Status Protocol (OCSP) Stapling, is enabled and configured. | ❌ Gap |  | Managed certificate only; no OCSP stapling or revocation rule. |
| V12.1.5 | L3 | Encrypted Client Hello (ECH) is enabled in the application's TLS settings to prevent exposure of sensitive metadata, such as the Server Name Indication (SNI), during TLS handshake processes. | ❌ Gap |  | No Encrypted Client Hello rule. |

### V12.2 HTTPS Communication with External Facing Services

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V12.2.1 | L1 | TLS is used for all connectivity between a client and external facing, HTTP-based services, and does not fall back to insecure or unencrypted communications. | ✅ Covered | [29 §1.2](../29-hardening-that-costs-nothing.md#12-the-rest-of-the-edge-all-free), [15 §8](../15-infrastructure-and-environments.md#8-static-content-and-content-delivery) | HTTP-to-HTTPS redirect at edge, transport-security header on every response. |
| V12.2.2 | L1 | External facing services use publicly trusted TLS certificates. | ✅ Covered | [29 §1.2](../29-hardening-that-costs-nothing.md#12-the-rest-of-the-edge-all-free), [15 §8](../15-infrastructure-and-environments.md#8-static-content-and-content-delivery) | Managed publicly-issued certificate per distribution, auto-renewed. |

### V12.3 General Service to Service Communication Security

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V12.3.1 | L2 | An encrypted protocol such as TLS is used for all inbound and outbound connections to and from the application, including monitoring systems, management tools, remote access and SSH, middleware, databases, mainframes, partner systems, or external APIs. The server must not fall back to insecure or unencrypted protocols. | ❌ Gap |  | No rule requiring encryption in transit for internal, database, or monitoring connections. |
| V12.3.2 | L2 | TLS clients validate certificates received before communicating with a TLS server. | ❌ Gap |  | No rule that backend TLS clients validate server certificates. |
| V12.3.3 | L2 | TLS or another appropriate transport encryption mechanism used for all connectivity between internal, HTTP-based services within the application, and does not fall back to insecure or unencrypted communications. | ❌ Gap |  | No rule on TLS between internal HTTP services. |
| V12.3.4 | L2 | TLS connections between internal services use trusted certificates. Where internally generated or self-signed certificates are used, the consuming service must be configured to only trust specific internal CAs and specific self-signed certificates. | ❌ Gap |  | No rule on internal CA trust or pinning. |
| V12.3.5 | L3 | Services communicating internally within a system (intra-service communications) use strong authentication to ensure that each endpoint is verified. Strong authentication methods, such as TLS client authentication, must be employed to ensure identity, using public-key infrastructure and mechanisms that are resistant to replay attacks. For microservice architectures, consider using a service mesh to simplify certificate management and enhance security. | 🟡 Partial | [18 §5](../18-security-and-privacy.md#5-least-privilege), [29 §5](../29-hardening-that-costs-nothing.md#5-the-account-and-the-pipeline) | One cloud identity per deployable; no mutual authentication rule for intra-service calls. |

## V13 Configuration

### V13.1 Configuration Documentation

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V13.1.1 | L2 | All communication needs for the application are documented. This must include external services which the application relies upon and cases where an end user might be able to provide an external location to which the application will then connect. | ❌ Gap |  | Infra is in code, but no documented inventory of communication needs or user-supplied destinations. |
| V13.1.2 | L3 | For each service the application uses, the documentation defines the maximum number of concurrent connections (e.g., connection pool limits) and how the application behaves when that limit is reached, including any fallback or recovery mechanisms, to prevent denial of service conditions. | ❌ Gap |  | No documented connection limits or behaviour at the limit per service. |
| V13.1.3 | L3 | The application documentation defines resource‑management strategies for every external system or service it uses (e.g., databases, file handles, threads, HTTP connections). This should include resource‑release procedures, timeout settings, failure handling, and where retry logic is implemented, specifying retry limits, delays, and back‑off algorithms. For synchronous HTTP request–response operations it should mandate short timeouts and either disable retries or strictly limit retries to prevent cascading delays and resource exhaustion. | 🟡 Partial | [06 §5.1](../06-api-contract-and-codegen.md#51-the-transport-core), [09 §2](../09-frontend.md#2-calling-the-api), [19 §4](../19-performance-and-scale.md#4-retries-backoff-and-caps) | Client timeouts and retry caps required; no per-system resource-management documentation. |
| V13.1.4 | L3 | The application's documentation defines the secrets that are critical for the security of the application and a schedule for rotating them, based on the organization's threat model and business requirements. | 🟡 Partial | [18 §6](../18-security-and-privacy.md#6-secrets), [15 §6](../15-infrastructure-and-environments.md#6-secrets) | Rotation procedure documented per secret; no criticality list or rotation schedule. |

### V13.2 Backend Communication Configuration

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V13.2.1 | L2 | Communications between backend application components that don't support the application's standard user session mechanism, including APIs, middleware, and data layers, are authenticated. Authentication must use individual service accounts, short-term tokens, or certificate-based authentication and not unchanging credentials such as passwords, API keys, or shared accounts with privileged access. | ✅ Covered | [18 §5](../18-security-and-privacy.md#5-least-privilege), [29 §5](../29-hardening-that-costs-nothing.md#5-the-account-and-the-pipeline), [28 §7](../28-supply-chain-and-build-inputs.md#7-the-pipeline-holds-no-long-lived-credential) | Role identity per deployable/function, short-lived federated credentials; no shared secrets. |
| V13.2.2 | L2 | Communications between backend application components, including local or operating system services, APIs, middleware, and data layers, are performed with accounts assigned the least necessary privileges. | ✅ Covered | [18 §5](../18-security-and-privacy.md#5-least-privilege), [22 §1](../22-pitfalls-checklist.md#1-new-permissions-required), [29 §5](../29-hardening-that-costs-nothing.md#5-the-account-and-the-pipeline), [31 §1](../31-multi-tenancy.md#1-the-rules) | Least-privilege role per function, named actions and resources, tenant-prefix scoping (R12). |
| V13.2.3 | L2 | If a credential has to be used for service authentication, the credential being used by the consumer is not a default credential (e.g., root/root or admin/admin). | 🟡 Partial | [18 §5](../18-security-and-privacy.md#5-least-privilege), [18 §6](../18-security-and-privacy.md#6-secrets) | Role identities replace credentials; no explicit ban on default service credentials. |
| V13.2.4 | L2 | An allowlist is used to define the external resources or systems with which the application is permitted to communicate (e.g., for outbound requests, data loads, or file access). This allowlist can be implemented at the application layer, web server, firewall, or a combination of different layers. | ❌ Gap |  | No outbound destination allowlist (only tool-server endpoint and redirect-URI allowlists). |
| V13.2.5 | L2 | The web or application server is configured with an allowlist of resources or systems to which the server can send requests or load data or files from. | ❌ Gap |  | No server-level egress allowlist. |
| V13.2.6 | L3 | Where the application connects to separate services, it follows the documented configuration for each connection, such as maximum parallel connections, behavior when maximum allowed connections is reached, connection timeouts, and retry strategies. | 🟡 Partial | [19 §4](../19-performance-and-scale.md#4-retries-backoff-and-caps), [06 §5.1](../06-api-contract-and-codegen.md#51-the-transport-core) | Retry caps/backoff and timeouts required; no per-connection documented limits. |

### V13.3 Secret Management

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V13.3.1 | L2 | A secrets management solution, such as a key vault, is used to securely create, store, control access to, and destroy backend secrets. These could include passwords, key material, integrations with databases and third-party systems, keys and seeds for time-based tokens, other internal secrets, and API keys. Secrets must not be included in application source code or included in build artifacts. For an L3 application, this must involve a hardware-backed solution such as an HSM. | 🟡 Partial | [18 §6](../18-security-and-privacy.md#6-secrets), [15 §6](../15-infrastructure-and-environments.md#6-secrets) | Secrets from parameter store or pipeline secret store; lifecycle (create/destroy) not specified. |
| V13.3.2 | L2 | Access to secret assets adheres to the principle of least privilege. | ✅ Covered | [31 §1](../31-multi-tenancy.md#1-the-rules), [29 §5](../29-hardening-that-costs-nothing.md#5-the-account-and-the-pipeline), [22 §1](../22-pitfalls-checklist.md#1-new-permissions-required) | No credential reaching another tenant (R13/R14); grants name exact resources. |
| V13.3.3 | L3 | All cryptographic operations are performed using an isolated security module (such as a vault or hardware security module) to securely manage and protect key material from exposure outside of the security module. | 🟡 Partial | [27 §6.4](../27-tool-servers-and-mcp.md#64-the-rules-that-make-the-rest-of-it-safe), [29 §7](../29-hardening-that-costs-nothing.md#7-what-is-not-free) | Customer-managed keys for tokens; not all crypto operations required in a security module. |
| V13.3.4 | L3 | Secrets are configured to expire and be rotated based on the application's documentation. | 🟡 Partial | [18 §6](../18-security-and-privacy.md#6-secrets), [28 §7](../28-supply-chain-and-build-inputs.md#7-the-pipeline-holds-no-long-lived-credential) | Rotation procedure and short-lived pipeline creds; no expiry/rotation schedule for app secrets. |

### V13.4 Unintended Information Leakage

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V13.4.1 | L1 | The application is deployed either without any source control metadata, including the .git or .svn folders, or in a way that these folders are inaccessible both externally and to the application itself. | ❌ Gap |  | No rule keeping .git or source-control metadata out of deployed bundles. |
| V13.4.2 | L2 | Debug modes are disabled for all components in production environments to prevent exposure of debugging features and information leakage. | 🟡 Partial | [18 §9](../18-security-and-privacy.md#9-things-that-quietly-become-security-surfaces), [27 §6.4](../27-tool-servers-and-mcp.md#64-the-rules-that-make-the-rest-of-it-safe) | Dev-only shims and test routes kept out of shipped entrypoint; no general debug-mode rule. |
| V13.4.3 | L2 | Web servers do not expose directory listings to clients unless explicitly intended. | 🟡 Partial | [29 §1.2](../29-hardening-that-costs-nothing.md#12-the-rest-of-the-edge-all-free), [15 §8](../15-infrastructure-and-environments.md#8-static-content-and-content-delivery) | Buckets private behind origin access control; directory listing not explicitly addressed. |
| V13.4.4 | L2 | Using the HTTP TRACE method is not supported in production environments, to avoid potential information leakage. | 🟡 Partial | [06 §1](../06-api-contract-and-codegen.md#1-the-transport) | API is RPC over POST only; TRACE not explicitly disabled at edge/gateway. |
| V13.4.5 | L2 | Documentation (such as for internal APIs) and monitoring endpoints are not exposed unless explicitly intended. | ❌ Gap |  | No rule hiding internal API docs (manifest) or monitoring endpoints. |
| V13.4.6 | L3 | The application does not expose detailed version information of backend components. | ❌ Gap |  | No rule suppressing backend version headers or banners. |
| V13.4.7 | L3 | The web tier is configured to only serve files with specific file extensions to prevent unintentional information, configuration, and source code leakage. | ❌ Gap |  | No rule restricting served file extensions. |

## V14 Data Protection

### V14.1 Data Protection Documentation

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V14.1.1 | L2 | All sensitive data created and processed by the application has been identified and classified into protection levels. This includes data that is only encoded and therefore easily decoded, such as Base64 strings or the plaintext payload inside a JWT. Protection levels need to take into account any data protection and privacy regulations and standards which the application is required to comply with. | 🟡 Partial | [18 §3](../18-security-and-privacy.md#3-personal-data), [06 §6.2](../06-api-contract-and-codegen.md#62-the-privacy-table-decides-what-leaves-the-building) | Required per-endpoint personal-data flag; binary, no protection levels or full data inventory. |
| V14.1.2 | L2 | All sensitive data protection levels have a documented set of protection requirements. This must include (but not be limited to) requirements related to general encryption, integrity verification, retention, how the data is to be logged, access controls around sensitive data in logs, database-level encryption, privacy and privacy-enhancing technologies to be used, and other confidentiality requirements. | 🟡 Partial | [18 §3](../18-security-and-privacy.md#3-personal-data), [17 §5](../17-observability-and-errors.md#5-logging), [29 §7](../29-hardening-that-costs-nothing.md#7-what-is-not-free) | Rules for personal data and tokens (logging, URLs, CMK); no per-level requirement set. |

### V14.2 General Data Protection

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V14.2.1 | L1 | Sensitive data is only sent to the server in the HTTP message body or header fields, and that the URL and query string do not contain sensitive information, such as an API key or session token. | ✅ Covered | [18 §1](../18-security-and-privacy.md#1-authentication), [18 §3](../18-security-and-privacy.md#3-personal-data), [06 §5.1](../06-api-contract-and-codegen.md#51-the-transport-core) | Tokens in headers only; personal data never in URL or query string. |
| V14.2.2 | L2 | The application prevents sensitive data from being cached in server components, such as load balancers and application caches, or ensures that the data is securely purged after use. | 🟡 Partial | [19 §6](../19-performance-and-scale.md#6-caching) | CDN caches only fingerprinted assets; no explicit rule against caching sensitive API data. |
| V14.2.3 | L2 | Defined sensitive data is not sent to untrusted parties (e.g., user trackers) to prevent unwanted collection of data outside of the application's control. | 🟡 Partial | [03 §3.3.2](../03-backend-domain-and-ports.md#332-the-client-instance-identifier), [17 §1](../17-observability-and-errors.md#1-one-reporter-everywhere), [09 §10](../09-frontend.md#10-errors-and-observability-in-the-client) | No personal data in error reports or analytics exports; no general tracker rule. |
| V14.2.4 | L2 | Controls around sensitive data related to encryption, integrity verification, retention, how the data is to be logged, access controls around sensitive data in logs, privacy and privacy-enhancing technologies, are implemented as defined in the documentation for the specific data's protection level. | 🟡 Partial | [18 §3](../18-security-and-privacy.md#3-personal-data), [17 §5](../17-observability-and-errors.md#5-logging), [29 §3](../29-hardening-that-costs-nothing.md#3-data) | Privacy gate, log bans, encryption defaults implemented; no documented-level mapping. |
| V14.2.5 | L3 | Caching mechanisms are configured to only cache responses which have the expected content type for that resource and do not contain sensitive, dynamic content. The web server should return a 404 or 302 response when a non-existent file is accessed rather than returning a different, valid file. This should prevent Web Cache Deception attacks. | 🟡 Partial | [19 §6](../19-performance-and-scale.md#6-caching), [15 §8](../15-infrastructure-and-environments.md#8-static-content-and-content-delivery) | Only fingerprinted assets cached; missing files return SPA shell 200, not 404. |
| V14.2.6 | L3 | The application only returns the minimum required sensitive data for the application's functionality. For example, only returning some of the digits of a credit card number and not the full number. If the complete data is required, it should be masked in the user interface unless the user specifically views it. | ✅ Covered | [18 §2](../18-security-and-privacy.md#2-authorization), [18 §3](../18-security-and-privacy.md#3-personal-data), [19 §7](../19-performance-and-scale.md#7-payload-size) | Return only fields a screen displays; reference by identifier; per-role response types. |
| V14.2.7 | L3 | Sensitive information is subject to data retention classification, ensuring that outdated or unnecessary data is deleted automatically, on a defined schedule, or as the situation requires. | 🟡 Partial | [31 §8](../31-multi-tenancy.md#8-lifecycle-r26-to-r30), [29 §3](../29-hardening-that-costs-nothing.md#3-data), [17 §5](../17-observability-and-errors.md#5-logging) | Delete on retention policy, TTLs, log retention; no retention classification for all data. |
| V14.2.8 | L3 | Sensitive information is removed from the metadata of user-submitted files unless storage is consented to by the user. | ❌ Gap |  | No rule stripping metadata (EXIF etc.) from uploaded files. |

### V14.3 Client-side Data Protection

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V14.3.1 | L1 | Authenticated data is cleared from client storage, such as the browser DOM, after the client or session is terminated. The 'Clear-Site-Data' HTTP response header field may be able to help with this but the client-side should also be able to clear up if the server connection is not available when the session is terminated. | 🟡 Partial | [18 §1](../18-security-and-privacy.md#1-authentication), [06 §5.1](../06-api-contract-and-codegen.md#51-the-transport-core) | Credential cleared on 401; no rule clearing client data on sign-out. |
| V14.3.2 | L2 | The application sets sufficient anti-caching HTTP response header fields (i.e., Cache-Control: no-store) so that sensitive data is not cached in browsers. | 🟡 Partial | [27 §6.3](../27-tool-servers-and-mcp.md#63-the-consent-page) | no-store only on the MCP consent page; not required on sensitive API responses. |
| V14.3.3 | L2 | Data stored in browser storage (such as localStorage, sessionStorage, IndexedDB, or cookies) does not contain sensitive data, with the exception of session tokens. | 🟡 Partial | [18 §1](../18-security-and-privacy.md#1-authentication), [03 §3.3.1](../03-backend-domain-and-ports.md#331-cookies-sessions-and-client-held-state) | Token in local storage allowed; no rule barring other sensitive data in browser storage. |

## V15 Secure Coding and Architecture

### V15.1 Secure Coding and Architecture Documentation

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V15.1.1 | L1 | Application documentation defines risk based remediation time frames for 3rd party component versions with vulnerabilities and for updating libraries in general, to minimize the risk from these components. | 🟡 Partial | [18 §8](../18-security-and-privacy.md#8-dependencies), [28 §4](../28-supply-chain-and-build-inputs.md#4-advisories-and-where-a-package-came-from) | Scheduled vuln report treated as work, new advisories fail; no remediation time frames. |
| V15.1.2 | L2 | An inventory catalog, such as software bill of materials (SBOM), is maintained of all third-party libraries in use, including verifying that components come from pre-defined, trusted, and continually maintained repositories. | 🟡 Partial | [28 §4](../28-supply-chain-and-build-inputs.md#4-advisories-and-where-a-package-came-from), [28 §6](../28-supply-chain-and-build-inputs.md#6-container-and-binary-inputs-are-pinned-by-digest-or-hash), [18 §8](../18-security-and-privacy.md#8-dependencies) | Pinned lock files, registry-only sources, hashes; no SBOM. |
| V15.1.3 | L2 | The application documentation identifies functionality which is time-consuming or resource-demanding. This must include how to prevent a loss of availability due to overusing this functionality and how to avoid a situation where building a response takes longer than the consumer's timeout. Potential defenses may include asynchronous processing, using queues, and limiting parallel processes per user and per application. | 🟡 Partial | [26 §5](../26-staged-tool-payloads.md#5-cost-ceilings), [19 §9](../19-performance-and-scale.md#9-cost-as-a-performance-property) | Cost ceilings documented for anonymous staging; not a general inventory of heavy functions. |
| V15.1.4 | L3 | Application documentation highlights third-party libraries which are considered to be "risky components". | 🟡 Partial | [28 §1.3](../28-supply-chain-and-build-inputs.md#13-what-the-agent-checks-before-it-proposes-a-package), [28 §2](../28-supply-chain-and-build-inputs.md#2-install-scripts-are-off-everywhere) | Package vetting and install-script allowlist with reasons; no risky-component list. |
| V15.1.5 | L3 | Application documentation highlights parts of the application where "dangerous functionality" is being used. | 🟡 Partial | [18 §9](../18-security-and-privacy.md#9-things-that-quietly-become-security-surfaces) | Table of surfaces that become security surfaces; not a full dangerous-functionality inventory. |

### V15.2 Security Architecture and Dependencies

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V15.2.1 | L1 | The application only contains components which have not breached the documented update and remediation time frames. | 🟡 Partial | [28 §4](../28-supply-chain-and-build-inputs.md#4-advisories-and-where-a-package-came-from), [18 §8](../18-security-and-privacy.md#8-dependencies) | Blocks newly added high/critical advisories; existing ones only listed, no deadline. |
| V15.2.2 | L2 | The application has implemented defenses against loss of availability due to functionality which is time-consuming or resource-demanding, based on the documented security decisions and strategies for this. | ✅ Covered | [29 §1.1](../29-hardening-that-costs-nothing.md#11-the-flat-rate-plan-is-the-big-one), [29 §1.2](../29-hardening-that-costs-nothing.md#12-the-rest-of-the-edge-all-free), [18 §4](../18-security-and-privacy.md#4-input-handling), [26 §5](../26-staged-tool-payloads.md#5-cost-ceilings), [19 §1](../19-performance-and-scale.md#1-pagination-is-decided-when-the-endpoint-is-written) | Edge rate rules, gateway throttling, size limits, cost ceilings, mandatory pagination. |
| V15.2.3 | L2 | The production environment only includes functionality that is required for the application to function, and does not expose extraneous functionality such as test code, sample snippets, and development functionality. | 🟡 Partial | [18 §9](../18-security-and-privacy.md#9-things-that-quietly-become-security-surfaces), [27 §6.4](../27-tool-servers-and-mcp.md#64-the-rules-that-make-the-rest-of-it-safe), [07 §6](../07-composition-and-entrypoints.md#6-local-entrypoint) | Dev shims and test routes kept out of deployed entrypoint; no general extraneous-code rule. |
| V15.2.4 | L3 | Third-party components and all of their transitive dependencies are included from the expected repository, whether internally owned or an external source, and that there is no risk of a dependency confusion attack. | 🟡 Partial | [28 §4](../28-supply-chain-and-build-inputs.md#4-advisories-and-where-a-package-came-from), [28 §1.3](../28-supply-chain-and-build-inputs.md#13-what-the-agent-checks-before-it-proposes-a-package), [28 §6](../28-supply-chain-and-build-inputs.md#6-container-and-binary-inputs-are-pinned-by-digest-or-hash) | Registry-only resolution, exact-name check, hashes; dependency confusion not named. |
| V15.2.5 | L3 | The application implements additional protections around parts of the application which are documented as containing "dangerous functionality" or using third-party libraries considered to be "risky components". This could include techniques such as sandboxing, encapsulation, containerization or network level isolation to delay and deter attackers who compromise one part of an application from pivoting elsewhere in the application. | 🟡 Partial | [28 §2](../28-supply-chain-and-build-inputs.md#2-install-scripts-are-off-everywhere), [27 §7](../27-tool-servers-and-mcp.md#7-what-a-tool-server-must-never-do) | Install scripts off, unprivileged tool server; no general isolation of risky code. |

### V15.3 Defensive Coding

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V15.3.1 | L1 | The application only returns the required subset of fields from a data object. For example, it should not return an entire data object, as some individual fields should not be accessible to users. | ✅ Covered | [04 §3.3](../04-use-cases.md#33-map-domain-to-contract-before-returning), [06 §4](../06-api-contract-and-codegen.md#4-contract-rules-the-extractor-enforces), [18 §2](../18-security-and-privacy.md#2-authorization), [18 §3](../18-security-and-privacy.md#3-personal-data) | Entities never returned; per-endpoint mapped contracts; per-role response types. |
| V15.3.2 | L2 | Where the application backend makes calls to external URLs, it is configured to not follow redirects unless it is intended functionality. | ❌ Gap |  | No rule on following redirects for backend outbound calls. |
| V15.3.3 | L2 | The application has countermeasures to protect against mass assignment attacks by limiting allowed fields per controller and action, e.g., it is not possible to insert or update a field value when it was not intended to be part of that action. | ✅ Covered | [06 §4](../06-api-contract-and-codegen.md#4-contract-rules-the-extractor-enforces), [04 §4](../04-use-cases.md#4-contract-type-rules), [04 §3.3](../04-use-cases.md#33-map-domain-to-contract-before-returning) | Dedicated request type per endpoint, no shared contracts, explicit mapping in use case. |
| V15.3.4 | L2 | All proxying and middleware components transfer the user's original IP address correctly using trusted data fields that cannot be manipulated by the end user, and the application and web server use this correct value for logging and security decisions such as rate limiting, taking into account that even the original IP address may not be reliable due to dynamic IPs, VPNs, or corporate firewalls. | 🟡 Partial | [27 §6.4](../27-tool-servers-and-mcp.md#64-the-rules-that-make-the-rest-of-it-safe) | Trust one proxy hop / signed CDN header, stated only for tool-server rate limits. |
| V15.3.5 | L2 | The application explicitly ensures that variables are of the correct type and performs strict equality and comparator operations. This is to avoid type juggling or type confusion vulnerabilities caused by the application code making an assumption about a variable type. | 🟡 Partial | [04 §3.1](../04-use-cases.md#31-validation-lives-here), [22 §3](../22-pitfalls-checklist.md#3-build-and-typecheck-after-a-backend-change), [12 §4](../12-testing-integration-and-guards.md#4-verify-the-product-not-the-exit-code) | Typed contracts, transport rejects wrong shape, typecheck gate; no strict-equality rule. |
| V15.3.6 | L2 | JavaScript code is written in a way that prevents prototype pollution, for example, by using Set() or Map() instead of object literals. | ❌ Gap |  | No prototype-pollution rule. |
| V15.3.7 | L2 | The application has defenses against HTTP parameter pollution attacks, particularly if the application framework makes no distinction about the source of request parameters (query string, body parameters, cookies, or header fields). | 🟡 Partial | [06 §1](../06-api-contract-and-codegen.md#1-the-transport), [07 §2](../07-composition-and-entrypoints.md#2-building-the-request-scope) | Single JSON POST body, identity only from verified token; duplicate parameters not addressed. |

### V15.4 Safe Concurrency

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V15.4.1 | L3 | Shared objects in multi-threaded code (such as caches, files, or in-memory objects accessed by multiple threads) are accessed safely by using thread-safe types and synchronization mechanisms like locks or semaphores to avoid race conditions and data corruption. | 🟡 Partial | [08 §4.1](../08-branches-and-data-scoping.md#41-explicit-never-ambient), [17 §6](../17-observability-and-errors.md#6-the-root-exception-handler) | Bans ambient async-local state; per-invocation report scope; no general thread-safety rule. |
| V15.4.2 | L3 | Checks on a resource's state, such as its existence or permissions, and the actions that depend on them are performed as a single atomic operation to prevent time-of-check to time-of-use (TOCTOU) race conditions. For example, checking if a file exists before opening it, or verifying a user’s access before granting it. | 🟡 Partial | [05 §3.1](../05-repositories-and-persistence.md#31-document--key-value-stores) | Conditional writes for create-if-absent/optimistic update; not stated for all check-then-act. |
| V15.4.3 | L3 | Locks are used consistently to avoid threads getting stuck, whether by waiting on each other or retrying endlessly, and that locking logic stays within the code responsible for managing the resource to ensure locks cannot be inadvertently or maliciously modified by external classes or code. | ❌ Gap |  | No rule on lock usage or deadlock avoidance. |
| V15.4.4 | L3 | Resource allocation policies prevent thread starvation by ensuring fair access to resources, such as by leveraging thread pools, allowing lower-priority threads to proceed within a reasonable timeframe. | ❌ Gap |  | No thread-starvation or fair resource allocation rule. |

## V16 Security Logging and Error Handling

### V16.1 Security Logging Documentation

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V16.1.1 | L2 | An inventory exists documenting the logging performed at each layer of the application's technology stack, what events are being logged, log formats, where that logging is stored, how it is used, how access to it is controlled, and for how long logs are kept. | 🟡 Partial | [17 §5](../17-observability-and-errors.md#5-logging), [29 §5](../29-hardening-that-costs-nothing.md#5-the-account-and-the-pipeline), [29 §6](../29-hardening-that-costs-nothing.md#6-the-tests-that-keep-them-on) | Structured logs with retention on every group; no logging inventory or access-control doc. |

### V16.2 General Logging

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V16.2.1 | L2 | Each log entry includes necessary metadata (such as when, where, who, what) that would allow for a detailed investigation of the timeline when an event happens. | 🟡 Partial | [17 §6](../17-observability-and-errors.md#6-the-root-exception-handler), [17 §1](../17-observability-and-errors.md#1-one-reporter-everywhere), [17 §5](../17-observability-and-errors.md#5-logging) | Request id on every line, site tag, structured fields; who/when not mandated. |
| V16.2.2 | L2 | Time sources for all logging components are synchronized, and that timestamps in security event metadata use UTC or include an explicit time zone offset. UTC is recommended to ensure consistency across distributed systems and to prevent confusion during daylight saving time transitions. | ❌ Gap |  | No rule on synchronized time or UTC log timestamps. |
| V16.2.3 | L2 | The application only stores or broadcasts logs to the files and services that are documented in the log inventory. | 🟡 Partial | [17 §1](../17-observability-and-errors.md#1-one-reporter-everywhere) | One reporter to one monitoring service; no documented destination list. |
| V16.2.4 | L2 | Logs can be read and correlated by the log processor that is in use, preferably by using a common logging format. | ✅ Covered | [17 §5](../17-observability-and-errors.md#5-logging) | Structured logging, one event per line with named fields. |
| V16.2.5 | L2 | When logging sensitive data, the application enforces logging based on the data's protection level. For example, it may not be allowed to log certain data, such as credentials or payment details. Other data, such as session tokens, may only be logged by being hashed or masked, either in full or partially. | ✅ Covered | [17 §5](../17-observability-and-errors.md#5-logging), [17 §1](../17-observability-and-errors.md#1-one-reporter-everywhere), [18 §1](../18-security-and-privacy.md#1-authentication), [18 §3](../18-security-and-privacy.md#3-personal-data) | Never log credentials, tokens, full bodies, or personal data; identifiers only. |

### V16.3 Security Events

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V16.3.1 | L2 | All authentication operations are logged, including successful and unsuccessful attempts. Additional metadata, such as the type of authentication or factors used, should also be collected. | ❌ Gap |  | No rule logging sign-in success/failure; delegated to identity service with no logging requirement. |
| V16.3.2 | L2 | Failed authorization attempts are logged. For L3, this must include logging all authorization decisions, including logging when sensitive data is accessed (without logging the sensitive data itself). | ❌ Gap |  | 17 §8 marks 401/403 as not reported; only operator cross-tenant reads audited (31 R20). |
| V16.3.3 | L2 | The application logs the security events that are defined in the documentation and also logs attempts to bypass the security controls, such as input validation, business logic, and anti-automation. | 🟡 Partial | [29 §1.1](../29-hardening-that-costs-nothing.md#11-the-flat-rate-plan-is-the-big-one) | Firewall logs on paid tier capture blocked requests; no defined security-event list. |
| V16.3.4 | L2 | The application logs unexpected errors and security control failures such as backend TLS failures. | ✅ Covered | [17 §6](../17-observability-and-errors.md#6-the-root-exception-handler), [17 §2](../17-observability-and-errors.md#2-silent-recovery-is-banned), [07 §3](../07-composition-and-entrypoints.md#3-dispatch-and-authorization) | Root handler reports every unexpected throw with request id; no silent catches. |

### V16.4 Log Protection

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V16.4.1 | L2 | All logging components appropriately encode data to prevent log injection. | 🟡 Partial | [17 §5](../17-observability-and-errors.md#5-logging) | Structured logging reduces injection; no explicit encoding rule. |
| V16.4.2 | L2 | Logs are protected from unauthorized access and cannot be modified. | ❌ Gap |  | No rule on log access control or immutability. |
| V16.4.3 | L2 | Logs are securely transmitted to a logically separate system for analysis, detection, alerting, and escalation. The aim is to ensure that if the application is breached, the logs are not compromised. | 🟡 Partial | [17 §1](../17-observability-and-errors.md#1-one-reporter-everywhere) | Errors go to external monitoring service; logs stay in the app account. |

### V16.5 Error Handling

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V16.5.1 | L2 | A generic message is returned to the consumer when an unexpected or security-sensitive error occurs, ensuring no exposure of sensitive internal system data such as stack traces, queries, secret keys, and tokens. | ✅ Covered | [17 §6](../17-observability-and-errors.md#6-the-root-exception-handler), [17 §8](../17-observability-and-errors.md#8-the-error-taxonomy-on-the-wire), [07 §7](../07-composition-and-entrypoints.md#7-response-helpers) | 500 body carries request id only; exception message never leaves the process. |
| V16.5.2 | L2 | The application continues to operate securely when external resource access fails, for example, by using patterns such as circuit breakers or graceful degradation. | 🟡 Partial | [19 §4](../19-performance-and-scale.md#4-retries-backoff-and-caps), [10 §3](../10-mobile.md#3-sync) | Capped retries with backoff, offline sync; no circuit-breaker or degradation rule. |
| V16.5.3 | L2 | The application fails gracefully and securely, including when an exception occurs, preventing fail-open conditions such as processing a transaction despite errors resulting from validation logic. | ✅ Covered | [06 §6.3](../06-api-contract-and-codegen.md#63-therefore-generate-at-deploy-time-and-fail-closed), [17 §2](../17-observability-and-errors.md#2-silent-recovery-is-banned), [04 §3.1](../04-use-cases.md#31-validation-lives-here), [31 §1](../31-multi-tenancy.md#1-the-rules) | Fail closed on missing tables/config, early return on validation, no silent recovery. |
| V16.5.4 | L3 | A "last resort" error handler is defined which will catch all unhandled exceptions. This is both to avoid losing error details that must go to log files and to ensure that an error does not take down the entire application process, leading to a loss of availability. | ✅ Covered | [17 §6](../17-observability-and-errors.md#6-the-root-exception-handler), [17 §9](../17-observability-and-errors.md#9-client-side) | One root handler per entrypoint, process-level backstop hooks, client error boundary. |

## V17 WebRTC

### V17.1 TURN Server

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V17.1.1 | L2 | The Traversal Using Relays around NAT (TURN) service only allows access to IP addresses that are not reserved for special purposes (e.g., internal networks, broadcast, loopback). Note that this applies to both IPv4 and IPv6 addresses. | ➖ N/A |  | Architecture has no WebRTC, TURN, media or signaling server. |
| V17.1.2 | L3 | The Traversal Using Relays around NAT (TURN) service is not susceptible to resource exhaustion when legitimate users attempt to open a large number of ports on the TURN server. | ➖ N/A |  | Architecture has no WebRTC, TURN, media or signaling server. |

### V17.2 Media

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V17.2.1 | L2 | The key for the Datagram Transport Layer Security (DTLS) certificate is managed and protected based on the documented policy for management of cryptographic keys. | ➖ N/A |  | Architecture has no WebRTC, TURN, media or signaling server. |
| V17.2.2 | L2 | The media server is configured to use and support approved Datagram Transport Layer Security (DTLS) cipher suites and a secure protection profile for the DTLS Extension for establishing keys for the Secure Real-time Transport Protocol (DTLS-SRTP). | ➖ N/A |  | Architecture has no WebRTC, TURN, media or signaling server. |
| V17.2.3 | L2 | Secure Real-time Transport Protocol (SRTP) authentication is checked at the media server to prevent Real-time Transport Protocol (RTP) injection attacks from leading to either a Denial of Service condition or audio or video media insertion into media streams. | ➖ N/A |  | Architecture has no WebRTC, TURN, media or signaling server. |
| V17.2.4 | L2 | The media server is able to continue processing incoming media traffic when encountering malformed Secure Real-time Transport Protocol (SRTP) packets. | ➖ N/A |  | Architecture has no WebRTC, TURN, media or signaling server. |
| V17.2.5 | L3 | The media server is able to continue processing incoming media traffic during a flood of Secure Real-time Transport Protocol (SRTP) packets from legitimate users. | ➖ N/A |  | Architecture has no WebRTC, TURN, media or signaling server. |
| V17.2.6 | L3 | The media server is not susceptible to the "ClientHello" Race Condition vulnerability in Datagram Transport Layer Security (DTLS) by checking if the media server is publicly known to be vulnerable or by performing the race condition test. | ➖ N/A |  | Architecture has no WebRTC, TURN, media or signaling server. |
| V17.2.7 | L3 | Any audio or video recording mechanisms associated with the media server are able to continue processing incoming media traffic during a flood of Secure Real-time Transport Protocol (SRTP) packets from legitimate users. | ➖ N/A |  | Architecture has no WebRTC, TURN, media or signaling server. |
| V17.2.8 | L3 | The Datagram Transport Layer Security (DTLS) certificate is checked against the Session Description Protocol (SDP) fingerprint attribute, terminating the media stream if the check fails, to ensure the authenticity of the media stream. | ➖ N/A |  | Architecture has no WebRTC, TURN, media or signaling server. |

### V17.3 Signaling

| ID | L | Requirement | Repo rule | Rules | Note |
|---|---|---|---|---|---|
| V17.3.1 | L2 | The signaling server is able to continue processing legitimate incoming signaling messages during a flood attack. This should be achieved by implementing rate limiting at the signaling level. | ➖ N/A |  | Architecture has no WebRTC, TURN, media or signaling server. |
| V17.3.2 | L2 | The signaling server is able to continue processing legitimate signaling messages when encountering malformed signaling message that could cause a denial of service condition. This could include implementing input validation, safely handling integer overflows, preventing buffer overflows, and employing other robust error-handling techniques. | ➖ N/A |  | Architecture has no WebRTC, TURN, media or signaling server. |

