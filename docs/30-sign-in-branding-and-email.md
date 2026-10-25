# 30 — Sign-in branding and transactional email

A managed identity service gives the project accounts, a sign-in page, and the emails that
carry a reset code or an invitation. Out of the box, all three carry the **provider's**
branding, and the email comes from the provider's address with a low daily cap.

Almost every project wants the same three changes, and they are usually left until someone
complains:

1. the sign-in page looks like the application;
2. the sign-in page is on the application's own domain;
3. the emails look like the application's other emails, and come from its own domain.

This file is the working template for all three. The code has **no dependencies**: the
handler imports nothing, and the email is a template literal. That is deliberate —
[28-supply-chain-and-build-inputs](28-supply-chain-and-build-inputs.md) § 1 makes a new
package a decision, and none of this needs one.

The service names are Reference A's (Amazon Cognito, SES, Terraform). The swap point is
the provider; every trap below exists in some form on all of them.

---

## 1. Decide these three before you write anything

| Question | Answer that costs least |
|---|---|
| Style the sign-in page, or build your own? | Style the hosted page. A hand-built sign-in page means the application handles passwords, which is the thing the identity service exists to avoid ([18-security-and-privacy](18-security-and-privacy.md) § 1). |
| Change the email text, or send the email yourself? | Change the text (§ 4). Sending it yourself needs a cryptography library and makes a failure mean **no email at all** (§ 6). |
| Provider's sender, or your own domain? | Your own domain, through the mail service (§ 3). The provider's default sender has a low daily cap and is not your address. |

---

## 2. The sign-in page

### 2.1 Branding is per app client — this is the trap

The customisation resource is keyed on **user pool and app client**. A pool with three
clients needs three resources. A client with none falls back to the stock provider styling,
and nobody notices until a user reaches that one flow.

Reference A has two: the web application's client, and the tool server's client. They share
the CSS and differ only in the logo, so a user landing on the tool-server consent flow can
see which sign-in they are completing.

### 2.2 Terraform

```hcl
# cognito/ui-customization.tf
#
# The classic hosted UI accepts a STRICT ALLOW-LIST of selectors and properties.
# Anything outside it is rejected by the API, with a message that names the
# selector. Build the CSS by adding one rule at a time, not by pasting a
# stylesheet.

locals {
  hosted_ui_css = <<-CSS
    .background-customizable   { background-color: #eef5ff; }
    .banner-customizable       { background-color: #ffffff; padding: 24px 16px 16px 16px; }
    .logo-customizable         { max-width: 60%; max-height: 60%; }

    .submitButton-customizable        { width: 100%; height: 44px; background-color: #0c4d99; color: #ffffff; font-size: 15px; font-weight: 700; margin: 8px 0 0 0; }
    .submitButton-customizable:hover  { background-color: #2f7de1; color: #ffffff; }

    .inputField-customizable       { width: 100%; height: 44px; background-color: #ffffff; border: 1px solid rgba(12, 87, 168, 0.2); color: #0d2540; }
    .inputField-customizable:focus { border-color: #0c57a8; outline: 2px; }

    .label-customizable          { color: #48627d; font-weight: 500; }
    .textDescription-customizable { display: block; padding: 8px 0; font-size: 14px; color: #48627d; }
    .idpDescription-customizable  { display: block; padding: 8px 0; font-size: 14px; color: #48627d; }
    .redirect-customizable        { color: #0c57a8; }
    .legalText-customizable       { color: #48627d; font-size: 12px; }

    .idpButton-customizable       { width: 100%; height: 40px; text-align: center; margin-bottom: 8px; color: #0d2540; background-color: rgba(255,255,255,0.8); border-color: rgba(12,87,168,0.2); }
    .idpButton-customizable:hover { color: #0d2540; background-color: #ffffff; }

    .errorMessage-customizable { width: 100%; margin: 8px 0; padding: 8px; background: #fef2f2; border: 1px solid #ef4444; color: #991b1b; box-sizing: border-box; }
  CSS
}

resource "aws_cognito_user_pool_ui_customization" "web" {
  user_pool_id = aws_cognito_user_pool.this.id
  client_id    = aws_cognito_user_pool_client.web.id

  css        = local.hosted_ui_css
  image_file = filebase64("${path.module}/assets/logo.png")

  # The pool must HAVE a domain before it can be customised.
  depends_on = [aws_cognito_user_pool_domain.this]
}

resource "aws_cognito_user_pool_ui_customization" "tool_server" {
  user_pool_id = aws_cognito_user_pool.this.id
  client_id    = aws_cognito_user_pool_client.tool_server.id

  css        = local.hosted_ui_css
  image_file = filebase64("${path.module}/assets/logo-tool-server.png")

  depends_on = [aws_cognito_user_pool_domain.this]
}
```

Rules:

- **`depends_on` the domain.** Customising a pool that has no domain fails.
- **The logo is base64 of a file in the repository**, so the branding is in version control
  and not uploaded by hand.
- **The allow-list is the design constraint.** No web fonts, no layout changes, no new
  elements. Match the colours and accept the layout, or move to § 2.4.
- A later branding system from the same provider (a designer with a settings document
  instead of CSS) usually sits on a **paid feature tier**, while the classic CSS
  customisation sits on the free one. Check which tier the account is on before planning
  around it ([29-hardening-that-costs-nothing](29-hardening-that-costs-nothing.md) § 7).

### 2.3 The sign-in page on your own domain

The sign-in page on the provider's domain tells the user they have left the application.
A custom domain fixes that, and it is three resources plus a DNS record.

```hcl
# cognito/custom-domain.tf
#
# The provider fronts the custom domain with its own CDN distribution, so the
# certificate must be in the CDN's certificate region (us-east-1), not the
# application's region. `aws.acm` is an aliased provider pinned to us-east-1.

resource "aws_acm_certificate" "auth" {
  count    = var.custom_domain != null ? 1 : 0
  provider = aws.acm

  domain_name       = var.custom_domain # e.g. auth.example.com
  validation_method = "DNS"

  lifecycle { create_before_destroy = true }
}

resource "aws_route53_record" "auth_cert_validation" {
  for_each = var.custom_domain != null ? {
    for option in aws_acm_certificate.auth[0].domain_validation_options :
    option.domain_name => {
      name   = option.resource_record_name
      record = option.resource_record_value
      type   = option.resource_record_type
    }
  } : {}

  allow_overwrite = true
  zone_id         = var.hosted_zone_id
  name            = each.value.name
  type            = each.value.type
  records         = [each.value.record]
  ttl             = 60
}

resource "aws_acm_certificate_validation" "auth" {
  count    = var.custom_domain != null ? 1 : 0
  provider = aws.acm

  certificate_arn         = aws_acm_certificate.auth[0].arn
  validation_record_fqdns = [for record in aws_route53_record.auth_cert_validation : record.fqdn]
}

resource "aws_route53_record" "auth" {
  count   = var.custom_domain != null ? 1 : 0
  zone_id = var.hosted_zone_id
  name    = var.custom_domain
  type    = "A"

  # The CDN's hosted zone id is a published constant, the same in every account.
  alias {
    name                   = aws_cognito_user_pool_domain.this.cloudfront_distribution
    zone_id                = "Z2FDTNDATAQYW2"
    evaluate_target_health = false
  }
}
```

Traps:

- **The certificate region is the CDN's region, not the application's.** A certificate in
  the application region is refused with a message that does not say why.
- **The parent domain must already resolve** before the custom domain can be created. On a
  brand-new zone, create something at the parent first.
- **Changing the domain of a live pool signs everybody out.** Treat it as a user-visible
  change, not as a rename ([15-infrastructure-and-environments](15-infrastructure-and-environments.md) § 4.1).

### 2.4 When the hosted page is not enough

If the allow-list genuinely cannot carry the brand, the next step is **not** a hand-built
sign-in form. Use the provider's own API from your own page only when you accept that your
code now handles credentials, and state that decision in a record
([20-decision-records](20-decision-records.md)). Reference A did not do this, and the
classic CSS has been enough for two years.

---

## 3. Sending from your own domain

The provider's default email sender exists for development. It has a **low daily cap** and
sends from an address the user does not recognise. Move to the mail service as soon as a
real user can receive an email.

```hcl
# cognito/main.tf (extract)

resource "aws_cognito_user_pool" "this" {
  name = "${var.name_prefix}-users"

  # With no mail identity configured, the pool keeps the provider's own sender.
  # That is the local and test configuration, and nothing else.
  dynamic "email_configuration" {
    for_each = var.ses_email_identity_arn != null ? [1] : []
    content {
      email_sending_account = "DEVELOPER"
      source_arn            = var.ses_email_identity_arn # the VERIFIED identity
      from_email_address    = var.ses_from_email_address # no-reply@example.com
    }
  }

  dynamic "lambda_config" {
    for_each = var.custom_message_lambda_arn != null ? [1] : []
    content {
      custom_message = var.custom_message_lambda_arn
    }
  }
}
```

- `source_arn` is the **verified identity**, and `from_email_address` must be inside it.
- The mail service starts in a sandbox that only sends to verified addresses. Leaving the
  sandbox is a support request, and it is the step people forget until production.
- Publish SPF, DKIM and a `p=reject` DMARC policy. Those are DNS records and cost nothing
  ([29-hardening-that-costs-nothing](29-hardening-that-costs-nothing.md) § 4).
- Both settings are **optional in the module**. A local or test stack with neither keeps
  the provider's sender and its default text, so the same module serves every environment.

---

## 4. The email text: the custom message trigger

The provider calls one function **before it sends any email**, and sends the subject and
body the function returns. The function receives the code as a **placeholder**, never as
the real value, so it needs no secret, no key, and no cryptography.

This is the zero-dependency path, and it is the right default.

### 4.1 The handler

No imports. Node built-ins only, and not even those.

```js
// cognito-custom-message.js  —  handler: cognito-custom-message.handler
//
// The identity provider calls this before it sends any email and sends what
// this returns. Five rules, each one from a real failure:
//
// 1. It waits 5 SECONDS for the whole invocation. A slow answer sends NOTHING.
// 2. FAIL OPEN. On any error, return the event UNCHANGED: the provider then
//    sends its own plain text. A plain reset code still lets the user sign in;
//    a thrown error sends no email at all.
// 3. `codeParameter` and `usernameParameter` are PLACEHOLDERS ({####},
//    {username}). The provider substitutes the real values AFTER this returns,
//    and REFUSES a message that lost a placeholder. Never trim, escape, or
//    re-case them.
// 4. The body has a hard length limit. Over it, the provider refuses the whole
//    message and the user gets nothing.
// 5. The event carries the user's email address and attributes. NEVER log the
//    event. Report the error only.

'use strict';

/** The provider refuses an email message longer than this. */
const MESSAGE_LIMIT = 20000;

/** Fallback name when the account carries no tenant name. */
const PRODUCT_NAME = 'Example Platform';

function escapeHtml(value) {
  return String(value)
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;');
}

// ── The shell ────────────────────────────────────────────────────────────────
// Light background with dark text, everywhere. Several mail clients apply a
// dark-mode transform that rewrites a background colour and leaves an inline
// text colour alone, so white-on-dark becomes white-on-light and the text
// disappears. Nothing below depends on its background surviving.

function shell(title, bodyHtml) {
  return `<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<meta name="color-scheme" content="light only">
<meta name="supported-color-schemes" content="light only">
<title>${escapeHtml(title)}</title>
<style>
  body { margin:0; padding:0; background-color:#eef5ff; color:#0d2540;
         font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',Arial,sans-serif; }
  .wrapper { max-width:620px; margin:32px auto; padding:0 16px 32px; }
  .card   { background:#ffffff; border:1px solid #c8d8ec; border-radius:20px; overflow:hidden; }
  .header { background-color:#f4f8fd; color:#0d2540; padding:32px 36px 28px;
            text-align:center; border-bottom:3px solid #0c4d99; }
  .header h1 { margin:0; font-family:Georgia,'Times New Roman',serif; font-size:24px; }
  .body   { padding:32px 36px; line-height:1.7; color:#35516d; }
  .body h2 { color:#0d2540; font-size:18px; margin:0 0 16px; }
  .body p  { margin:0 0 14px; }
  /* The code, large and alone. A dark border keeps it a box if the fill is dropped. */
  .code   { font-family:'Courier New',Courier,monospace; font-size:32px; font-weight:700;
            letter-spacing:0.3em; text-align:center; color:#0d2540; background-color:#f4f8fd;
            border:2px solid #0c4d99; border-radius:12px; padding:16px 8px; margin:20px 0; }
  table   { width:100%; border-collapse:collapse; margin:0 0 16px; }
  th      { background:#f4f8fd; border:1px solid #c8d8ec; padding:10px 14px; text-align:left;
            font-size:13px; color:#0d2540; }
  td      { border:1px solid #c8d8ec; padding:10px 14px; font-size:14px; color:#35516d; }
  .note   { font-size:13px; color:#64748b; }
  .footer { background:#f4f8fd; padding:22px 36px; border-top:1px solid #c8d8ec;
            text-align:center; font-size:12px; color:#4a5a70; }
</style>
</head>
<body>
  <div class="wrapper">
    <div class="card">
      <div class="header"><h1>${escapeHtml(title)}</h1></div>
      <div class="body">${bodyHtml}</div>
      <div class="footer">This message was sent automatically. Do not reply to it.</div>
    </div>
  </div>
</body>
</html>`;
}

// ── One entry per trigger source ─────────────────────────────────────────────
// A source that is NOT in this map keeps the provider's own text. That is the
// correct answer for a source you have not designed an email for.

const TEMPLATES = {
  CustomMessage_ForgotPassword: (request, account) => ({
    subject: `Your ${account} password reset code`,
    html: shell('Reset your password', `
      <h2>Reset your password</h2>
      <p>Use this code to set a new password for your
         <strong>${escapeHtml(account)}</strong> account:</p>
      <p class="code">${request.codeParameter}</p>
      <p class="note">If you did not ask for this code, ignore this email.
         Do not give the code to anyone.</p>`),
  }),

  CustomMessage_AdminCreateUser: (request, account) => ({
    subject: `Your ${account} account`,
    html: shell('Your account is ready', `
      <h2>Your account is ready</h2>
      <p>An account has been created for you on
         <strong>${escapeHtml(account)}</strong>. You choose your own password
         the first time you sign in.</p>
      <table>
        <tr><th>Username</th><td>${request.usernameParameter}</td></tr>
        <tr><th>Temporary password</th><td>${request.codeParameter}</td></tr>
      </table>
      <p class="note">The temporary password expires. If it does, ask an
         administrator to send a new one.</p>`),
  }),

  CustomMessage_SignUp: (request, account) => ({
    subject: `Confirm your ${account} account`,
    html: shell('Confirm your account', `
      <h2>Confirm your account</h2>
      <p>Enter this code to confirm your
         <strong>${escapeHtml(account)}</strong> account:</p>
      <p class="code">${request.codeParameter}</p>`),
  }),

  CustomMessage_ResendCode: (request, account) => ({
    subject: `Your new ${account} confirmation code`,
    html: shell('Your new confirmation code', `
      <h2>Your new confirmation code</h2>
      <p>You asked for a new code for your
         <strong>${escapeHtml(account)}</strong> account:</p>
      <p class="code">${request.codeParameter}</p>`),
  }),

  CustomMessage_Authentication: (request, account) => ({
    subject: `Your ${account} sign-in code`,
    html: shell('Your sign-in code', `
      <h2>Your sign-in code</h2>
      <p>Enter this code to finish signing in to your
         <strong>${escapeHtml(account)}</strong> account:</p>
      <p class="code">${request.codeParameter}</p>
      <p class="note">If you are not signing in, ignore this email.</p>`),
  }),

  CustomMessage_VerifyUserAttribute: (request, account) => ({
    subject: 'Confirm your email address',
    html: shell('Confirm your email address', `
      <h2>Confirm your email address</h2>
      <p>Enter this code to confirm this address on your
         <strong>${escapeHtml(account)}</strong> account:</p>
      <p class="code">${request.codeParameter}</p>`),
  }),

  CustomMessage_UpdateUserAttribute: (request, account) => ({
    subject: 'Confirm your new email address',
    html: shell('Confirm your new email address', `
      <h2>Confirm your new email address</h2>
      <p>Enter this code to confirm the new address on your
         <strong>${escapeHtml(account)}</strong> account:</p>
      <p class="code">${request.codeParameter}</p>`),
  }),
};

/**
 * Pure. Unit-test THIS, not the handler.
 * Returns the event, changed or unchanged. It never throws.
 */
function applyCustomMessage(event, reportError) {
  try {
    const template = TEMPLATES[event.triggerSource];
    if (!template) return event;

    const attributes = event.request.userAttributes || {};
    const account = attributes['custom:tenantName'] || PRODUCT_NAME;
    const message = template(event.request, account);

    if (message.html.length > MESSAGE_LIMIT) {
      reportError(new Error(
        `${event.triggerSource} message is ${message.html.length} characters; the limit is ${MESSAGE_LIMIT}.`,
      ));
      return event;
    }

    event.response.emailSubject = message.subject;
    event.response.emailMessage = message.html;
    return event;
  } catch (error) {
    reportError(error);
    return event;
  }
}

// The message only. The event holds the user's address.
function reportError(error) {
  console.error('[custom-message]', error instanceof Error ? error.message : String(error));
}

exports.handler = async (event) => applyCustomMessage(event, reportError);
exports.applyCustomMessage = applyCustomMessage;
exports.TEMPLATES = TEMPLATES;
exports.MESSAGE_LIMIT = MESSAGE_LIMIT;
```

### 4.2 The tenant name, and why it is an attribute

The account name in the email comes from a **user attribute**, not from a database read.

A read inside this trigger is a real risk: the invocation has 5 seconds for everything,
and the storage client retries a slow or throttled read with backoff. Reference A does read
a row, and had to give the read its **own** 1.5-second cap and a fallback name, because the
alternative is a user who gets no reset code at all.

**Prefer the attribute.** If the name must be read, copy this shape:

- give the read a cap well under the trigger's budget;
- on timeout, failure or a missing row, use the neutral product name and **send the email**;
- never let the header be the reason an email does not arrive.

### 4.3 Terraform for the trigger

```hcl
# api/cognito-custom-message.tf
#
# ORDER MATTERS. The pool reads this function's ARN, and the permission below
# reads the pool's ARN. Nothing here depends on the pool, so the modules do not
# form a cycle: function -> pool -> permission.

locals {
  custom_message_function_name = "cognito-custom-message-${var.environment}"
}

data "archive_file" "custom_message" {
  type        = "zip"
  source_file = var.custom_message_source_file # .../cognito-custom-message.js
  output_path = "${path.module}/build/cognito-custom-message.zip"
}

resource "aws_iam_role" "custom_message" {
  name = "cognito-custom-message-${var.environment}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Action    = "sts:AssumeRole"
      Principal = { Service = "lambda.amazonaws.com" }
    }]
  })
}

# Inline, not a managed-policy attachment: the managed-policy quota per role is
# small (15-infrastructure-and-environments § 4.2). This function writes logs
# and nothing else.
resource "aws_iam_role_policy" "custom_message" {
  role = aws_iam_role.custom_message.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["logs:CreateLogStream", "logs:PutLogEvents"]
      Resource = "${aws_cloudwatch_log_group.custom_message.arn}:*"
    }]
  })
}

# The NAME comes from the local, not from the function, so the function can
# depend on its log group without a cycle.
resource "aws_cloudwatch_log_group" "custom_message" {
  name              = "/aws/lambda/${local.custom_message_function_name}"
  retention_in_days = var.log_retention_days
}

resource "aws_lambda_function" "custom_message" {
  function_name    = local.custom_message_function_name
  filename         = data.archive_file.custom_message.output_path
  source_code_hash = data.archive_file.custom_message.output_base64sha256
  role             = aws_iam_role.custom_message.arn
  handler          = "cognito-custom-message.handler"
  runtime          = "nodejs22.x"

  # The provider waits 5 seconds. A longer timeout cannot help and hides the
  # real limit from whoever reads this next.
  timeout     = 5
  memory_size = 256

  depends_on = [aws_cloudwatch_log_group.custom_message]
}

# Without this the provider cannot call the function, and every email silently
# falls back to the default text.
resource "aws_lambda_permission" "cognito_invoke_custom_message" {
  statement_id  = "AllowCognitoInvokeCustomMessage"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.custom_message.function_name
  principal     = "cognito-idp.amazonaws.com"
  source_arn    = var.user_pool_arn
}
```

---

## 5. Tests

**Unit, on `applyCustomMessage`** — it is pure, so these are fast and need no cloud
([11-testing-unit](11-testing-unit.md)):

- one case per trigger source: the subject and the body are set;
- **every placeholder survives** — assert the body still contains `{####}`, and that the
  invitation body still contains `{username}`. This is the test that catches an "improved"
  template that escaped or trimmed them;
- an unknown trigger source returns the event **unchanged**;
- a body over the limit leaves the event unchanged and reports an error;
- a thrown template error leaves the event unchanged and reports an error;
- the account name falls back to the product name when the attribute is missing;
- the account name is **escaped**: an attribute of `<script>` does not reach the body raw;
- `reportError` is given the error, and nothing in the function receives the event.

**Guard tests** ([12-testing-integration-and-guards](12-testing-integration-and-guards.md)):

- every app client whose hosted page a human can reach has a customisation resource;
- the function's timeout is not above the provider's own wait;
- the invoke permission exists and its source is the pool;
- the pool's mail configuration is set in every deployed environment.

**Manual, once per trigger source.** No automated test proves the email **renders**. Send
each one to a real address, open it in a dark-mode client, and keep the screenshots with
the change ([18-security-and-privacy](18-security-and-privacy.md) § 10 is the same rule for
outbound links).

---

## 6. The custom email sender, and why it is the second choice

The provider also offers a trigger that hands the **whole send** to your function: you
receive the event and send the email yourself.

Take it only for a reason on this list:

- the email must be localised per user from your own content;
- the mail service's own templates, suppression lists or per-message tags are needed;
- the message must carry an attachment, or go over a channel that is not email.

What it costs, and both of these are the reason Reference A did not take it:

- **It is not zero dependency.** The code arrives encrypted with a key, in the
  envelope format of the provider's encryption SDK. It is **not** a plain key-service
  decrypt call, so decrypting it means adding that SDK — a new direct dependency, with the
  whole of [28-supply-chain-and-build-inputs](28-supply-chain-and-build-inputs.md) § 1
  attached to it.
- **It is a one-way door at runtime.** Once the trigger is set, the provider sends **no**
  email itself. A bug in your function is not a plain-text email — it is **no email**, so
  nobody can reset a password. The custom message trigger fails open; this one fails
  closed.

If you take it: give the function a key grant and nothing else, never log the decrypted
code, keep the send inside the invocation budget, and alarm on its error rate, because a
silent failure here locks users out.

---

## 7. Pitfalls

| Pitfall | What you see | Fix |
|---|---|---|
| Customisation on one client only | One flow is stock-branded | One resource per client (§ 2.1) |
| Customisation before the domain exists | Apply fails | `depends_on` the domain |
| CSS outside the allow-list | API refuses the property | Add one rule at a time |
| Certificate in the wrong region | Custom domain refuses the certificate | Issue it in the CDN's region |
| Parent domain does not resolve | Custom domain creation fails | Create a record at the parent first |
| Missing invoke permission | Emails are the default text, no error anywhere | The permission resource, with the pool as source |
| Placeholder removed or escaped | Provider refuses the message; no email | Assert the placeholders in a test |
| Body over the limit | Provider refuses the message; no email | Check the length and fail open |
| Reading a row inside the trigger | Occasional "no email arrived" | Attribute, or a hard cap plus a fallback |
| Logging the event | Email addresses in the logs | Log the error message only |
| Mail service still in the sandbox | Works for the team, not for users | Leave the sandbox before launch |
| White text on a dark panel | Text disappears in a dark-mode client | Light background, dark text, bordered boxes |

---

## 8. Adopting it

1. Add the pool with **no** mail configuration and **no** trigger. The provider's defaults
   work, and the local stack never needs more.
2. Add the hosted-page customisation, one resource per client, and look at every flow.
3. Verify the sending domain, publish SPF, DKIM and DMARC, and switch the pool to it.
4. Add the custom message trigger with one template, prove the placeholder survives, then
   add the rest.
5. Send one of each to a real address, in a dark-mode client, and keep the screenshots.
6. Leave the mail sandbox before the first real user.
