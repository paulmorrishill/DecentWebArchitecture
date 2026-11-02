# CIS AWS Foundations Benchmark checklist

The **CIS Amazon Web Services Foundations Benchmark v7.0.0** (published March 2026, announced in the CIS April 2026 update), with the **AWS Foundational Security Best Practices** (FSBP) v1.0.0 controls for the services this architecture uses. CIS lists v7.1.0 as its newest version as of November 2026; its recommendation list was not public to check, so confirm the numbering against it before an assessment. How to run the review is [33-security-review](../33-security-review.md).

The benchmark checks **AWS accounts**, not the application. Run every row once per account: pre-production, production, and the organisation's management and log accounts if there are any. The architecture is serverless — functions, a key/value store, buckets behind a distribution, a managed user pool — so the rows written for servers, file systems and relational databases are ➖ N/A, and each says when it would stop being N/A. Recommendations are numbered from section 2, as in v7.0.0.

- **Repo rule.** ✅ a rule here, if followed, meets the item. 🟡 a rule meets part of it. ❌ no rule. ➖ the architecture as written has no such surface. 🏢 an organisational, people or physical control that an architecture cannot cover. **A tick is not a pass.** It says where to look; the reviewer still checks.
- **Level and assessment.** L1 is the baseline, L2 is defence in depth. Automated rows can be checked by a tool; Manual rows need a person, though a tool may still help.
- **Evidence.** AWS Security Hub CSPM runs the CIS benchmark (v5.0.0, the newest version it supports) and FSBP in every account and region it is turned on in. Its findings export is acceptable evidence under rule 2 of [33](../33-security-review.md#1-the-rules) for every row whose note names a Hub control. A row whose note says "Hub: none", or a control marked "not in the CIS standard", needs other evidence: a screenshot of the setting, the infrastructure file and line, or the output of a CLI call.
- **Tracker.** [cis-aws.csv](cis-aws.csv) has the same rows with empty result columns.
- **Wording.** Items are paraphrased; the CIS text is licensed and is the authority. Only the recommendation numbers and the substance of the titles are used here.

---

## What a SaaS team gets wrong

- **Every account has its own root user.** One account per environment ([15](../15-infrastructure-and-environments.md) § 2) means root MFA, no root access key and a security contact on each one, or centralised root access (2.1.1), which removes the member accounts' root credentials. The management account's root stays yours to protect.
- **The trail belongs to the organisation, not the environment.** An organisation trail delivered to a separate log account cannot be stopped by an administrator of the account it watches. A trail inside production can. Every monitoring row in section 5 reads that trail.
- **"No VPC" is not "no network rows".** Every account has a default VPC in every region, with a network ACL that allows all inbound and a default security group that fails 6.5. Delete the default VPCs, and turn on EBS default encryption (6.1.1) anyway: it is a free account setting the check reads whether or not a volume exists. Only the rows about instances, file systems, relational databases and peering are truly N/A.
- **The regions you do not use count.** The trail, Config, Access Analyzer, Security Hub and GuardDuty are per region. Turn them on everywhere, or deny the unused regions with a service control policy (2.1.2).
- **A budget alert is not monitoring.** [15](../15-infrastructure-and-environments.md) § 7 alerts on spend. Section 5 wants alarms on root use, policy changes and console failures — fifteen metric filters on one log group, or the same events through an event rule.
- **The account password policy applies with no IAM users.** Security Hub reads it either way (2.8, 2.9). The policy in [29](../29-hardening-that-costs-nothing.md) § 2 is the product's user pool, not IAM's.
- **The free edge tier fails 4.10 by design.** A static site on the free plan has no access log ([29](../29-hardening-that-costs-nothing.md) § 1.1). Accept it in a decision record, or pay for the tier.
- **Security Hub checks v5.0.0, not v7.0.0.** The rows v6 and v7 added — Organizations (2.1.x), Principal "*" (2.21), web front-end logging (4.10), Organizations changes (5.15), Security Hub itself (5.16) — and every Manual row need other evidence.

## Coverage

| | ✅ Covered | 🟡 Partial | ❌ Gap | ➖ N/A | 🏢 Org | Total |
|---|---|---|---|---|---|---|
| 2 Identity and access management | 1 | 8 | 15 | 1 | 1 | 26 |
| 3 Storage | 1 | 2 | 1 | 5 | 0 | 9 |
| 4 Logging | 0 | 1 | 9 | 0 | 0 | 10 |
| 5 Monitoring | 0 | 0 | 16 | 0 | 0 | 16 |
| 6 Networking | 0 | 0 | 3 | 6 | 0 | 9 |
| **All** | 2 | 11 | 44 | 12 | 1 | 70 |
| FSBP supplement (not in All) | 8 | 22 | 7 | 1 | 0 | 38 |

Of the 70 recommendations, 41 are Level 1 and 29 Level 2; 34 are Automated and 36 Manual.

## 2 Identity and access management

| ID | Item | Repo rule | Rules | Note |
|---|---|---|---|---|
| 2.1.1 | Root credentials of member accounts managed centrally from AWS Organizations (L2, Manual) | ❌ Gap | [15 §2](../15-infrastructure-and-environments.md#2-environments), [29 §5](../29-hardening-that-costs-nothing.md#5-the-account-and-the-pipeline) | Hub: none. Per-environment accounts exist; no rule puts them in an Organization or removes member root credentials. |
| 2.1.2 | Organization-wide authorisation guardrails (service and resource control policies) on every account (L2, Manual) | ❌ Gap |  | Hub: none. Nothing denies unused regions, leaving the Organization or stopping the trail. |
| 2.1.3 | The Organizations management account runs no workloads (L2, Manual) | ❌ Gap | [15 §2](../15-infrastructure-and-environments.md#2-environments) | Hub: none. The set names staging and production accounts only; nothing keeps the payer account empty. |
| 2.1.4 | Organizational units laid out by environment and data sensitivity (L2, Manual) | 🟡 Partial | [15 §2](../15-infrastructure-and-environments.md#2-environments), [29 §5](../29-hardening-that-costs-nothing.md#5-the-account-and-the-pipeline), [31 R7](../31-multi-tenancy.md) | Hub: none. Environments are split by account; no OU layout, and silo customer accounts need a place too. |
| 2.1.5 | A delegated administrator, not the management account, manages Organizations policies (L2, Manual) | ❌ Gap |  | Hub: none. No rule. |
| 2.1.6 | Organization-integrated services administered from a delegated account (L2, Manual) | ❌ Gap |  | Hub: none. No account named to administer Security Hub, GuardDuty, Config or Access Analyzer. |
| 2.2 | Account contact details kept current (L1, Manual) | 🏢 Org |  | Hub: none. Contacts per account; an organisational process. |
| 2.3 | Security contact registered on each account (L1, Manual) | ❌ Gap | [15 §5](../15-infrastructure-and-environments.md#5-applying) | Hub Account.1. The alternate contact can be set in infrastructure code; no rule asks for it. |
| 2.4 | No access key exists for the root user (L1, Automated) | 🟡 Partial | [28 §7](../28-supply-chain-and-build-inputs.md#7-the-pipeline-holds-no-long-lived-credential), [29 §5](../29-hardening-that-costs-nothing.md#5-the-account-and-the-pipeline) | Hub IAM.4. No long-lived pipeline key; root is not named, and every member account has its own root. |
| 2.5 | MFA on the root user (L1, Automated) | ❌ Gap | [33 §6.1](../33-security-review.md#61-the-gaps-that-matter-most) | Hub IAM.9. Gap 3 of doc 33; once per account, not once per organisation. |
| 2.6 | Hardware MFA on the root user (L2, Manual) | ❌ Gap |  | Hub IAM.6. No rule; a security key held by a named custodian. |
| 2.7 | Root user not used for administration or daily work (L1, Manual) | 🟡 Partial | [15 §5](../15-infrastructure-and-environments.md#5-applying), [29 §5](../29-hardening-that-costs-nothing.md#5-the-account-and-the-pipeline) | Hub IAM.20 (FSBP only). Pipeline-only applies remove the need; no ban and no break-glass procedure. |
| 2.8 | Account IAM password policy requires at least 14 characters (L1, Automated) | ❌ Gap | [29 §2](../29-hardening-that-costs-nothing.md#2-identity) | Hub IAM.15. 29 §2 sets the user pool policy, not IAM's; set the account policy even with no IAM users. |
| 2.9 | Account IAM password policy prevents reuse (L1, Automated) | ❌ Gap | [29 §2](../29-hardening-that-costs-nothing.md#2-identity) | Hub IAM.16. As 2.8. |
| 2.10 | MFA for every IAM user with a console password (L1, Automated) | ❌ Gap | [33 §6.1](../33-security-review.md#61-the-gaps-that-matter-most) | Hub IAM.5. No rule requires MFA on staff accounts (CE-UAC-4). |
| 2.11 | Credentials unused for 45 days or more are disabled (L1, Automated) | ❌ Gap |  | Hub IAM.22. No leaver or inactivity rule (CE-UAC-3). |
| 2.12 | Access keys rotated within 90 days (L1, Automated) | 🟡 Partial | [28 §7](../28-supply-chain-and-build-inputs.md#7-the-pipeline-holds-no-long-lived-credential), [18 §6](../18-security-and-privacy.md#6-secrets) | Hub IAM.3. The pipeline holds none; human keys not banned; rotation is documented for secrets only. |
| 2.13 | IAM users get permissions only through groups (L1, Automated) | 🟡 Partial | [18 §5](../18-security-and-privacy.md#5-least-privilege), [29 §5](../29-hardening-that-costs-nothing.md#5-the-account-and-the-pipeline) | Hub IAM.2. One role per deployable; no rule bans IAM users or policies attached to them. |
| 2.14 | No attached policy grants full "*:*" administrative access (L1, Automated) | 🟡 Partial | [18 §5](../18-security-and-privacy.md#5-least-privilege), [29 §5](../29-hardening-that-costs-nothing.md#5-the-account-and-the-pipeline), [28 §7](../28-supply-chain-and-build-inputs.md#7-the-pipeline-holds-no-long-lived-credential) | Hub IAM.1 (FSBP only). Function roles are narrow; the deploy role's scope is unstated and no guard test checks it. |
| 2.15 | A support role exists to manage incidents with AWS Support (L1, Automated) | ❌ Gap |  | Hub IAM.18. No rule; one role in infrastructure code per account. |
| 2.16 | EC2 instances use instance roles for AWS access (L2, Automated) | ➖ N/A | [16 §2.4](../16-ci-cd.md#24-self-hosted-runners) | Hub: none. No EC2 in the design; applies if self-hosted runners run on EC2. |
| 2.17 | Expired TLS certificates removed from IAM (L1, Automated) | ✅ Covered | [29 §1.2](../29-hardening-that-costs-nothing.md#12-the-rest-of-the-edge-all-free), [30 §2.3](../30-sign-in-branding-and-email.md#23-the-sign-in-page-on-your-own-domain) | Hub IAM.26. Certificates are provider-managed and renewed; none is uploaded to IAM. |
| 2.18 | IAM Access Analyzer for external access on in every region (L1, Automated) | ❌ Gap |  | Hub IAM.28. No rule; per region per account, or one organisation analyser. |
| 2.19 | Human access managed centrally through federation or Organizations (L2, Manual) | 🟡 Partial | [28 §7](../28-supply-chain-and-build-inputs.md#7-the-pipeline-holds-no-long-lived-credential), [15 §5](../15-infrastructure-and-environments.md#5-applying) | Hub: none. The pipeline is federated; no rule for staff console access. |
| 2.20 | AWSCloudShellFullAccess restricted (L1, Manual) | ❌ Gap | [18 §5](../18-security-and-privacy.md#5-least-privilege) | Hub IAM.27. Least privilege covers deployables, not staff roles. |
| 2.21 | No resource policy grants unrestricted access with Principal "*" (L1, Manual) | 🟡 Partial | [29 §1.2](../29-hardening-that-costs-nothing.md#12-the-rest-of-the-edge-all-free), [29 §6](../29-hardening-that-costs-nothing.md#6-the-tests-that-keep-them-on), [18 §5](../18-security-and-privacy.md#5-least-privilege) | Hub S3.2, S3.3, Lambda.1, SQS.3, SNS.4, KMS.5. Guard test for buckets only; none for function, queue, topic or key policies. |

## 3 Storage

| ID | Item | Repo rule | Rules | Note |
|---|---|---|---|---|
| 3.1.1 | Bucket policies deny requests that do not use TLS (L2, Automated) | 🟡 Partial | [29 §1.2](../29-hardening-that-costs-nothing.md#12-the-rest-of-the-edge-all-free), [29 §3](../29-hardening-that-costs-nothing.md#3-data) | Hub S3.5. The edge redirects to HTTPS; no TLS-only deny on bucket policies, so a signed link works over HTTP. |
| 3.1.2 | MFA Delete on buckets (L2, Manual) | ❌ Gap | [29 §7](../29-hardening-that-costs-nothing.md#7-what-is-not-free) | Hub S3.20. Needs versioning (paid) and the root user to turn on; the pipeline cannot. Decide in a decision record. |
| 3.1.3 | Data in S3 discovered, classified and secured where needed (L2, Manual) | 🟡 Partial | [18 §3](../18-security-and-privacy.md#3-personal-data), [29 §3](../29-hardening-that-costs-nothing.md#3-data), [31 R30](../31-multi-tenancy.md) | Hub Macie.1 (FSBP only). Personal data declared per endpoint and per deletion list; nothing scans the buckets. |
| 3.1.4 | S3 Block Public Access on (L1, Automated) | ✅ Covered | [29 §3](../29-hardening-that-costs-nothing.md#3-data), [29 §1.2](../29-hardening-that-costs-nothing.md#12-the-rest-of-the-edge-all-free), [29 §6](../29-hardening-that-costs-nothing.md#6-the-tests-that-keep-them-on) | Hub S3.1, S3.8. Guard test per bucket; also set the account-level block in every account. |
| 3.2.1 | RDS encryption at rest (L1, Automated) | ➖ N/A | [31 R8](../31-multi-tenancy.md) | Hub RDS.3. No RDS in Reference A; a relational store (Reference B) on RDS makes 3.2.x apply. |
| 3.2.2 | RDS automatic minor version upgrade (L1, Automated) | ➖ N/A | [31 R8](../31-multi-tenancy.md) | Hub RDS.13. As 3.2.1. |
| 3.2.3 | RDS instances not publicly accessible (L1, Automated) | ➖ N/A | [31 R8](../31-multi-tenancy.md) | Hub RDS.2. As 3.2.1. |
| 3.2.4 | RDS Multi-AZ deployments (L1, Manual) | ➖ N/A | [31 R8](../31-multi-tenancy.md) | Hub RDS.5, RDS.15. As 3.2.1. |
| 3.3.1 | EFS encryption at rest (L1, Automated) | ➖ N/A |  | Hub EFS.1, EFS.8. No file system in the design. |

## 4 Logging

| ID | Item | Repo rule | Rules | Note |
|---|---|---|---|---|
| 4.1 | CloudTrail on in all regions (L1, Manual) | ❌ Gap | [33 §7](../33-security-review.md#7-other-things-worth-doing) | Hub CloudTrail.1. No account or organisation trail rule; 33 §7 names the benchmark as unmapped. |
| 4.2 | CloudTrail log file validation (L2, Automated) | ❌ Gap |  | Hub CloudTrail.4. Follows 4.1. |
| 4.3 | AWS Config on in all regions (L2, Automated) | ❌ Gap |  | Hub Config.1. Security Hub needs Config for most checks; no rule turns it on. |
| 4.4 | Server access logging on the CloudTrail bucket (L1, Manual) | ❌ Gap |  | Hub CloudTrail.7. Follows 4.1. |
| 4.5 | CloudTrail logs encrypted with a KMS customer-managed key (L2, Automated) | ❌ Gap | [29 §7](../29-hardening-that-costs-nothing.md#7-what-is-not-free) | Hub CloudTrail.2. 29 §7 keeps customer keys for named data; the trail is not on the list. |
| 4.6 | Rotation on customer-created symmetric KMS keys (L2, Automated) | ❌ Gap | [27 §6.4](../27-tool-servers-and-mcp.md#64-the-rules-that-make-the-rest-of-it-safe), [29 §7](../29-hardening-that-costs-nothing.md#7-what-is-not-free) | Hub KMS.4. Customer keys are required for upstream tokens; no rotation rule. |
| 4.7 | VPC flow logs in every VPC (L2, Automated) | ❌ Gap |  | Hub EC2.6. No VPC in the design, but every region has a default VPC; delete it or log it. |
| 4.8 | Object-level write events logged for buckets (L2, Automated) | ❌ Gap | [29 §7](../29-hardening-that-costs-nothing.md#7-what-is-not-free) | Hub S3.22. Data events cost money and 29 §7 does not list them. |
| 4.9 | Object-level read events logged for buckets (L2, Automated) | ❌ Gap | [29 §7](../29-hardening-that-costs-nothing.md#7-what-is-not-free) | Hub S3.23. As 4.8; who read a member's file is the first breach question. |
| 4.10 | Access logging on every AWS-managed web front end (L1, Manual) | 🟡 Partial | [29 §1.1](../29-hardening-that-costs-nothing.md#11-the-flat-rate-plan-is-the-big-one), [29 §7](../29-hardening-that-costs-nothing.md#7-what-is-not-free) | Hub CloudFront.5, APIGateway.1, APIGateway.9. Paid tier logs the API edge; free-tier static sites have no logs by design. |

## 5 Monitoring

| ID | Item | Repo rule | Rules | Note |
|---|---|---|---|---|
| 5.1 | Unauthorised API calls monitored (L2, Manual) | ❌ Gap | [33 §6.1](../33-security-review.md#61-the-gaps-that-matter-most), [17 §8](../17-observability-and-errors.md#8-the-error-taxonomy-on-the-wire) | Hub CloudWatch.2 (not in the CIS standard). Needs the trail in a log group; 15 §7 alerts on spend only. |
| 5.2 | Console sign-in without MFA monitored (L1, Manual) | ❌ Gap |  | Hub CloudWatch.3 (not in the CIS standard). Needs 4.1. |
| 5.3 | Root user activity monitored (L1, Manual) | ❌ Gap |  | Hub CloudWatch.1 (not in the CIS standard). The alarm that matters most with a root per account. |
| 5.4 | IAM policy changes monitored (L1, Manual) | ❌ Gap | [15 §5](../15-infrastructure-and-environments.md#5-applying) | Hub CloudWatch.4 (not in the CIS standard). Only the pipeline role should change policy; alarm on anyone else. |
| 5.5 | CloudTrail configuration changes monitored (L1, Manual) | ❌ Gap |  | Hub CloudWatch.5 (not in the CIS standard). Needs 4.1. |
| 5.6 | Console authentication failures monitored (L2, Manual) | ❌ Gap |  | Hub CloudWatch.6 (not in the CIS standard). Needs 4.1. |
| 5.7 | Disabling or scheduled deletion of customer-created KMS keys monitored (L2, Manual) | ❌ Gap | [27 §6.4](../27-tool-servers-and-mcp.md#64-the-rules-that-make-the-rest-of-it-safe) | Hub CloudWatch.7 (not in the CIS standard). Losing the token key loses the data it encrypts. |
| 5.8 | S3 bucket policy changes monitored (L1, Manual) | ❌ Gap |  | Hub CloudWatch.8 (not in the CIS standard). Needs 4.1. |
| 5.9 | AWS Config configuration changes monitored (L2, Manual) | ❌ Gap |  | Hub CloudWatch.9 (not in the CIS standard). Needs 4.1 and 4.3. |
| 5.10 | Security group changes monitored (L2, Manual) | ❌ Gap |  | Hub CloudWatch.10 (not in the CIS standard). No VPC is used, so any change is the signal. |
| 5.11 | Network ACL changes monitored (L2, Manual) | ❌ Gap |  | Hub CloudWatch.11 (not in the CIS standard). As 5.10. |
| 5.12 | Network gateway changes monitored (L1, Manual) | ❌ Gap |  | Hub CloudWatch.12 (not in the CIS standard). As 5.10. |
| 5.13 | Route table changes monitored (L1, Manual) | ❌ Gap |  | Hub CloudWatch.13 (not in the CIS standard). As 5.10. |
| 5.14 | VPC changes monitored (L1, Manual) | ❌ Gap |  | Hub CloudWatch.14 (not in the CIS standard). As 5.10. |
| 5.15 | AWS Organizations changes monitored (L1, Manual) | ❌ Gap |  | Hub: none. Watch from the management account; needs an organisation trail. |
| 5.16 | AWS Security Hub on (L2, Automated) | ❌ Gap | [33 §4](../33-security-review.md#4-the-automated-checks) | Hub: none. 33 §4 lists pipeline checks only; turn it on in every account and region. |

## 6 Networking

| ID | Item | Repo rule | Rules | Note |
|---|---|---|---|---|
| 6.1.1 | EBS default encryption on in every region (L1, Automated) | ❌ Gap | [29 §3](../29-hardening-that-costs-nothing.md#3-data) | Hub EC2.7. No volumes in the design, but the check reads a free per-region account setting. |
| 6.1.2 | CIFS reachable only from trusted networks (L1, Automated) | ➖ N/A |  | Hub: none. No instances and no security groups in the design. |
| 6.2 | No network ACL allows ingress from 0.0.0.0/0 to remote administration ports (L1, Automated) | ❌ Gap |  | Hub EC2.21. The default VPC's ACL allows all inbound; fails until default VPCs are deleted. |
| 6.3 | No security group allows ingress from 0.0.0.0/0 to remote administration ports (L1, Automated) | ➖ N/A |  | Hub EC2.53. Functions run outside a VPC; live the day one joins a VPC. |
| 6.4 | No security group allows ingress from ::/0 to remote administration ports (L1, Automated) | ➖ N/A |  | Hub EC2.54. As 6.3. |
| 6.5 | The default security group of every VPC restricts all traffic (L2, Automated) | ❌ Gap |  | Hub EC2.2. Default VPCs' default groups allow traffic; delete default VPCs or empty the group. |
| 6.6 | VPC peering routes are least access (L2, Manual) | ➖ N/A |  | Hub: none. No peering in the design. |
| 6.7 | EC2 metadata service allows IMDSv2 only (L1, Automated) | ➖ N/A | [16 §2.4](../16-ci-cd.md#24-self-hosted-runners) | Hub EC2.8. No EC2 in the design; applies to EC2-hosted runners. |
| 6.8 | VPC endpoints used to reach AWS services (L2, Manual) | ➖ N/A |  | Hub: none. No VPC; functions call services over public TLS endpoints. |

## AWS Foundational Security Best Practices: what CIS does not cover

CIS checks the account. These FSBP controls check the services the architecture is built from. The IDs are Security Hub control IDs; every one is Automated and runs in the FSBP standard. Controls already named in a CIS row above are not repeated.

| ID | Item | Repo rule | Rules | Note |
|---|---|---|---|---|
| Lambda.1 | Function resource policies allow no public invocation | 🟡 Partial | [18 §5](../18-security-and-privacy.md#5-least-privilege), [30 §4.3](../30-sign-in-branding-and-email.md#43-terraform-for-the-trigger) | Permissions name the calling service and source; no guard test. |
| Lambda.2 | Functions run on a supported runtime | ❌ Gap | [28 §4](../28-supply-chain-and-build-inputs.md#4-advisories-and-where-a-package-came-from), [33 §6.1](../33-security-review.md#61-the-gaps-that-matter-most) | Nothing retires an end-of-life runtime (gap 6 of doc 33). |
| APIGateway.1 | Execution logging on REST and WebSocket stages | 🟡 Partial | [17 §5](../17-observability-and-errors.md#5-logging), [29 §5](../29-hardening-that-costs-nothing.md#5-the-account-and-the-pipeline) | Retention required; execution logging not named. Full request logging would break 17 §5. |
| APIGateway.4 | Gateway associated with a WAF web ACL | 🟡 Partial | [29 §1.1](../29-hardening-that-costs-nothing.md#11-the-flat-rate-plan-is-the-big-one) | Firewall is on the distribution; turn off the default endpoint or the edge can be bypassed. |
| APIGateway.8 | Every route declares an authorisation type | ✅ Covered | [18 §1](../18-security-and-privacy.md#1-authentication), [18 §2](../18-security-and-privacy.md#2-authorization), [31 R15](../31-multi-tenancy.md) | Token verified at the gateway; anonymous routes show as NONE, so list them as evidence. |
| APIGateway.9 | Access logging on API Gateway v2 stages | 🟡 Partial | [17 §5](../17-observability-and-errors.md#5-logging), [29 §1.1](../29-hardening-that-costs-nothing.md#11-the-flat-rate-plan-is-the-big-one) | Edge logs on the paid tier; stage access log not required. |
| APIGateway.11 | Custom domain names use a recommended TLS security policy | 🟡 Partial | [29 §1.2](../29-hardening-that-costs-nothing.md#12-the-rest-of-the-edge-all-free) | Minimum TLS set at the edge; the gateway's own domain not named. |
| DynamoDB.2 | Point-in-time recovery on tables | 🟡 Partial | [29 §3](../29-hardening-that-costs-nothing.md#3-data), [29 §6](../29-hardening-that-costs-nothing.md#6-the-tests-that-keep-them-on), [29 §7](../29-hardening-that-costs-nothing.md#7-what-is-not-free) | Paid, so only where recovery is required; the guard test enforces that list. |
| DynamoDB.6 | Deletion protection on tables | ✅ Covered | [29 §3](../29-hardening-that-costs-nothing.md#3-data), [29 §6](../29-hardening-that-costs-nothing.md#6-the-tests-that-keep-them-on) | Guard test fails a table without it. |
| CloudFront.1 | Default root object configured | 🟡 Partial | [15 §8](../15-infrastructure-and-environments.md#8-static-content-and-content-delivery) | The client shell is served; no rule names the setting. |
| CloudFront.3 | Viewers must use HTTPS | ✅ Covered | [29 §1.2](../29-hardening-that-costs-nothing.md#12-the-rest-of-the-edge-all-free) | HTTP to HTTPS redirect at the edge. |
| CloudFront.5 | Distribution logging | 🟡 Partial | [29 §1.1](../29-hardening-that-costs-nothing.md#11-the-flat-rate-plan-is-the-big-one) | The API distribution on the paid tier logs; free-tier static sites cannot. |
| CloudFront.6 | WAF on every distribution | ✅ Covered | [29 §1.1](../29-hardening-that-costs-nothing.md#11-the-flat-rate-plan-is-the-big-one), [29 §6](../29-hardening-that-costs-nothing.md#6-the-tests-that-keep-them-on) | The flat-rate plan includes the ACL, even on the free tier. |
| CloudFront.12 | No origin points at a bucket that no longer exists | 🟡 Partial | [15 §8](../15-infrastructure-and-environments.md#8-static-content-and-content-delivery) | Bucket and distribution live in one module; no guard against a dangling origin. |
| CloudFront.13 | Origin access control on S3 origins | ✅ Covered | [29 §1.2](../29-hardening-that-costs-nothing.md#12-the-rest-of-the-edge-all-free), [29 §6](../29-hardening-that-costs-nothing.md#6-the-tests-that-keep-them-on) | Origin access control plus block public access. |
| CloudFront.15 | Recommended viewer TLS security policy | ✅ Covered | [29 §1.2](../29-hardening-that-costs-nothing.md#12-the-rest-of-the-edge-all-free) | Minimum TLS version set; record the policy name as evidence. |
| Cognito.2 | Identity pools allow no unauthenticated identities | ➖ N/A | [15 §1](../15-infrastructure-and-environments.md#1-layout) | The design has a user pool and no identity pool. |
| Cognito.3 | Strong password policy on user pools | 🟡 Partial | [29 §2](../29-hardening-that-costs-nothing.md#2-identity) | Length-based; the control's defaults also want character classes, so set its parameters. |
| Cognito.4 | Threat protection in full-function mode for custom authentication | ❌ Gap | [29 §7](../29-hardening-that-costs-nothing.md#7-what-is-not-free) | A paid feature plan, not listed in 29 §7. Decide in a decision record. |
| Cognito.5 | MFA on user pools | 🟡 Partial | [29 §2](../29-hardening-that-costs-nothing.md#2-identity), [33 §6.1](../33-security-review.md#61-the-gaps-that-matter-most) | One-time-code second factor offered, not required. |
| Cognito.6 | Deletion protection on user pools | 🟡 Partial | [15 §4](../15-infrastructure-and-environments.md#4-reviewing-a-change) | Plan review keeps pools out of the destroy list; no setting and no guard test. |
| S3.9 | Server access logging on buckets | ❌ Gap | [29 §7](../29-hardening-that-costs-nothing.md#7-what-is-not-free) | Not required anywhere; start with the buckets that hold member files. |
| S3.12 | No ACLs used to grant bucket access | 🟡 Partial | [29 §3](../29-hardening-that-costs-nothing.md#3-data) | Access is by policy and origin access control; no rule sets bucket-owner-enforced ownership. |
| S3.13 | Lifecycle configuration on buckets | 🟡 Partial | [31 R30](../31-multi-tenancy.md), [29 §3](../29-hardening-that-costs-nothing.md#3-data) | Deletion list exists; no lifecycle rule expires staged or abandoned objects. |
| KMS.1 | Customer managed IAM policies do not allow decrypt on all keys | 🟡 Partial | [29 §5](../29-hardening-that-costs-nothing.md#5-the-account-and-the-pipeline) | Grants name exact resources; no guard on decrypt with "*". |
| KMS.2 | Inline policies do not allow decrypt on all keys | 🟡 Partial | [29 §5](../29-hardening-that-costs-nothing.md#5-the-account-and-the-pipeline), [15 §4.2](../15-infrastructure-and-environments.md#42-quotas) | Inline policies are preferred, so this is the one that matters; no guard. |
| KMS.3 | Keys not scheduled for deletion by mistake | 🟡 Partial | [15 §4](../15-infrastructure-and-environments.md#4-reviewing-a-change) | Plan review protects state; keys are not on the never-destroy list. |
| KMS.5 | Key policies allow no public access | 🟡 Partial | [18 §5](../18-security-and-privacy.md#5-least-privilege) | No guard test over key policies. |
| WAF.10 | Web ACLs hold at least one rule or rule group | ✅ Covered | [29 §1.1](../29-hardening-that-costs-nothing.md#11-the-flat-rate-plan-is-the-big-one), [29 §6](../29-hardening-that-costs-nothing.md#6-the-tests-that-keep-them-on) | Five rules; the scanner guard test checks them. |
| WAF.12 | CloudWatch metrics on WAF rules | ❌ Gap |  | Not mentioned. |
| GuardDuty.1 | GuardDuty on | ❌ Gap | [33 §7](../33-security-review.md#7-other-things-worth-doing) | Threat detection named in 33 §7, no rule; per account and region. |
| GuardDuty.6 | GuardDuty Lambda Protection on | ❌ Gap |  | Watches function network activity; no rule. |
| Inspector.4 | Inspector standard scanning of Lambda functions | 🟡 Partial | [28 §4](../28-supply-chain-and-build-inputs.md#4-advisories-and-where-a-package-came-from) | Advisories checked at build; nothing rescans deployed functions. |
| SQS.1 | Queues encrypted at rest | ✅ Covered | [29 §3](../29-hardening-that-costs-nothing.md#3-data) | Default encryption with a provider-owned key. |
| SQS.3 | Queue policies allow no public access | 🟡 Partial | [29 §5](../29-hardening-that-costs-nothing.md#5-the-account-and-the-pipeline) | Grants name exact queues; no guard. |
| SNS.4 | Topic policies allow no public access | 🟡 Partial | [29 §5](../29-hardening-that-costs-nothing.md#5-the-account-and-the-pipeline) | As SQS.3. |
| SES.3 | Configuration sets require TLS for sending | ❌ Gap | [29 §4](../29-hardening-that-costs-nothing.md#4-email), [30 §3](../30-sign-in-branding-and-email.md#3-sending-from-your-own-domain) | SPF, DKIM and DMARC set; the TLS policy is not. |
| IAM.21 | Customer managed policies use no service-wide wildcard actions | 🟡 Partial | [18 §5](../18-security-and-privacy.md#5-least-privilege), [29 §5](../29-hardening-that-costs-nothing.md#5-the-account-and-the-pipeline) | Inline policies are preferred and this control does not read them; review those by hand. |

---

Sources: [CIS Amazon Web Services Benchmarks](https://www.cisecurity.org/benchmark/amazon_web_services) (lists v7.1.0 and v7.0.0); [CIS Benchmarks April 2026 Update](https://www.cisecurity.org/insights/blog/cis-benchmarks-april-2026-update) (v7.0.0); [Prowler CIS 7.0 for AWS mapping](https://github.com/prowler-cloud/prowler/blob/master/prowler/compliance/aws/cis_7.0_aws.json) (numbers, levels, assessment status); [Security Hub CSPM: CIS AWS Foundations Benchmark](https://docs.aws.amazon.com/securityhub/latest/userguide/cis-aws-foundations-benchmark.html) (control IDs, v5.0.0 mapping); [Security Hub CSPM: FSBP standard](https://docs.aws.amazon.com/securityhub/latest/userguide/fsbp-standard.html). Checked 2026-11-02.
