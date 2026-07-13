# Chapter 20 — Production Capstone Project: Enterprise Amazon S3 Platform (Production Edition)

**Difficulty:** Advanced (Course Capstone) — calibrated to a Cloud Engineer with 3–5 years of AWS production experience
**AWS Services Used:** S3, IAM, KMS, Organizations, Bucket Policies, Access Points & Multi-Region Access Points, Versioning, Object Lock, Lifecycle Rules, S3 Event Notifications, EventBridge, Lambda, SNS, SQS, Cross-Region/Cross-Account Replication, CloudWatch, CloudTrail, Config, GuardDuty (S3 Protection), Macie, Storage Lens, Terraform/CloudFormation
**Lab Type:** End-to-End Project with Infrastructure as Code
**Production Relevance:** ★★★★★

---

## What Changed From the Original Capstone

The original version of this project is a solid single-account, console-driven exercise. A 3–5 year engineer will be evaluated on things the original brief doesn't force you to confront:

| Original Assumption | Production Reality |
|---|---|
| One AWS account | Multi-account via AWS Organizations (dev/staging/prod + log-archive + security-tooling) |
| SSE-S3 default encryption | Customer-managed KMS keys (CMKs) with key policies and rotation |
| Console clicks | Infrastructure as Code (Terraform), reviewed and deployed via CI/CD |
| "Enable CloudTrail" | Org-wide CloudTrail to a dedicated, immutable log-archive account |
| "Configure Lifecycle Rules" | Cost-modeled lifecycle decisions backed by access-pattern data |
| Implicit trust in IAM users | SSO/IAM Identity Center, roles over users, permission boundaries |
| DR as an afterthought | RTO/RPO targets, tested runbooks, replication metrics (S3 RTC) |
| No compliance angle | Object Lock in Compliance mode, Macie for PII discovery, Config rules |

Everything below layers onto the original 12 phases — it doesn't replace them, it makes them defensible in a design review or an interview.

---

## 1. Account & Organization Structure

Before touching S3, decide *where* it lives.

```
Management Account (Organizations root)
 ├── Security-Tooling Account   → GuardDuty, Macie, Config aggregator, IAM Identity Center
 ├── Log-Archive Account        → CloudTrail org trail, S3 access logs, VPC flow logs (Object Lock enabled)
 ├── Shared-Services Account    → CI/CD pipeline, Terraform state bucket + DynamoDB lock table
 ├── Dev Account                → cloudmart-dev-* buckets
 ├── Staging Account             → cloudmart-staging-* buckets
 └── Prod Account                → cloudmart-* buckets (this capstone's primary scope)
```

**Why this matters in an interview:** a single-account design means an IAM misconfiguration or compromised credential in "dev" can touch production data. Account boundaries are a hard security control, not a convenience — IAM policy alone is not considered sufficient isolation for regulated data (financial reports, legal documents) in most real organizations.

**Task 0 (new):** Stand up the account structure (or simulate with separate OU tags if using a single sandbox account) and document the blast-radius boundary for each account.

---

## 2. Infrastructure as Code Baseline

Production S3 is never click-ops. Convert Phases 1–9 of the original brief into Terraform.

```hcl
# main.tf — example for one bucket; replicate module pattern for all 5
module "cloudmart_documents" {
  source      = "./modules/s3-bucket"
  bucket_name = "cloudmart-documents-${var.env}-${data.aws_caller_identity.current.account_id}"

  versioning_enabled   = true
  kms_key_arn          = module.kms.documents_key_arn
  block_public_access  = true
  enable_object_lock   = false

  lifecycle_rules = [
    {
      id     = "transition-and-expire"
      status = "Enabled"
      transitions = [
        { days = 30,  storage_class = "STANDARD_IA" },
        { days = 90,  storage_class = "GLACIER" },
        { days = 365, storage_class = "DEEP_ARCHIVE" }
      ]
      abort_incomplete_multipart_upload_days = 7
    }
  ]

  tags = {
    DataClassification = "Confidential"
    CostCenter          = "Finance"
    Owner               = "cloud-platform-team"
  }
}
```

Notes for the exercise:
- Bucket names must be globally unique — include account ID or a random suffix, never hardcode.
- State for this Terraform lives in its **own** bucket in the shared-services account, with versioning + DynamoDB state locking. Bootstrapping that bucket is the one piece of S3 infra you're allowed to click-ops, because Terraform can't create its own backend.
- Every module should emit `terraform plan` output as a PR artifact before `apply` — this is what a reviewer expects in a real PR.

**Deliverable:** a `modules/s3-bucket` Terraform module reused across all five buckets, plus a root module per environment.

---

## 3. Identity: Roles, Not Users

Replace the original Phase 2 IAM Users with roles assumed via IAM Identity Center (SSO) or federated OIDC (for CI/CD). Long-lived IAM users with access keys are treated as a finding in most production security reviews.

**Example least-privilege policy — Finance role, folder-scoped, with hard guardrails:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListOwnPrefixOnly",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::cloudmart-documents-prod",
      "Condition": {
        "StringLike": { "s3:prefix": ["finance/*"] }
      }
    },
    {
      "Sid": "ReadWriteFinancePrefix",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::cloudmart-documents-prod/finance/*",
      "Condition": {
        "Bool": { "aws:MultiFactorAuthPresent": "true" },
        "StringEquals": { "s3:x-amz-server-side-encryption": "aws:kms" }
      }
    },
    {
      "Sid": "DenyDeleteWithoutMFA",
      "Effect": "Deny",
      "Action": "s3:DeleteObject",
      "Resource": "arn:aws:s3:::cloudmart-documents-prod/finance/*",
      "Condition": {
        "BoolIfExists": { "aws:MultiFactorAuthPresent": "false" }
      }
    }
  ]
}
```

Key production details this adds over the original brief:
- **Condition-based scoping** (`s3:prefix`, `s3:x-amz-server-side-encryption`) instead of just resource ARNs.
- **MFA-gated deletes** as a compensating control instead of relying only on "MFA Delete" bucket versioning (which requires the bucket owner's root credentials to toggle and is operationally painful — know the trade-off and be able to explain why most teams use IAM conditions instead).
- **Permission boundaries** on any role that can create IAM entities (e.g., the CI/CD deploy role), capping what it can grant even if the policy attached to it is over-broad.

**Task (revised Phase 2):** Implement one role per department using this pattern, validate with `iam simulate-principal-policy`, and additionally run [IAM Access Analyzer](https://docs.aws.amazon.com/IAM/latest/UserGuide/what-is-access-analyzer.html) against the account to confirm no unintended external access exists.

---

## 4. Bucket Policies — Defense in Depth

The original brief says "deny public access, require HTTPS." Production policies also need to:

```json
{
  "Sid": "DenyUnEncryptedObjectUploads",
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:PutObject",
  "Resource": "arn:aws:s3:::cloudmart-documents-prod/*",
  "Condition": {
    "StringNotEquals": { "s3:x-amz-server-side-encryption": "aws:kms" }
  }
},
{
  "Sid": "DenyOldTLS",
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:*",
  "Resource": [
    "arn:aws:s3:::cloudmart-documents-prod",
    "arn:aws:s3:::cloudmart-documents-prod/*"
  ],
  "Condition": {
    "NumericLessThan": { "s3:TlsVersion": "1.2" }
  }
},
{
  "Sid": "RestrictToVPCEndpoint",
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:*",
  "Resource": [
    "arn:aws:s3:::cloudmart-documents-prod",
    "arn:aws:s3:::cloudmart-documents-prod/*"
  ],
  "Condition": {
    "StringNotEquals": { "aws:sourceVpce": "vpce-0123456789abcdef0" }
  }
}
```

Add a **Gateway VPC Endpoint for S3** so application traffic never traverses the public internet, and enforce it at the bucket policy level (`aws:sourceVpce`). This is a common interview probe: "how do you make sure your app servers reach S3 without going through NAT/IGW?"

---

## 5. Event-Driven Automation — Modernized

The original brief wires S3 Event Notifications directly to Lambda/SNS/SQS. That's fine for a single-consumer pattern, but production systems generally route through **EventBridge** so multiple, independently-owned consumers can subscribe without editing the bucket's notification config every time a new team needs the event.

```
S3 (EventBridge notifications enabled)
        │
        ▼
  Amazon EventBridge (default bus)
        │
   ┌────┴────┬─────────────┬───────────────┐
   ▼         ▼             ▼               ▼
 Lambda   SNS Topic     SQS Queue     Step Functions
(image     (finance      (log         (multi-step
resize)    approvals)    processing)   invoice workflow)
```

Requirements for this capstone section:

| Folder | Trigger | Consumer | Justification |
|---|---|---|---|
| `images/` | `Object Created` | Lambda (resize/thumbnail via Sharp or Pillow layer) | Synchronous, low-latency transform |
| `finance/` | `Object Created` | SNS → email/Slack | Human-in-the-loop approval notification |
| `logs/` | `Object Created` | SQS → batch processor | Decouples producer rate from consumer processing rate; supports retry/DLQ |
| `documents/` | `Object Created` | EventBridge → Step Functions | Multi-step (virus scan → classify via Macie → route) |

**Add a Dead Letter Queue (DLQ)** to every SQS integration and a **Lambda Destinations (on-failure)** target for the image-processing function. Grading criterion: can you show a failed event landing in the DLQ and explain your redrive strategy?

---

## 6. Lifecycle & Storage Class Strategy — Data-Driven, Not Guessed

Don't hardcode 30/90/365 days without justification. Production teams base transitions on **S3 Storage Class Analysis** and **Storage Lens** access-pattern data.

Revised strategy per bucket:

| Bucket | Access Pattern | Strategy |
|---|---|---|
| `cloudmart-images` | Frequent for 30 days, occasional after | **Intelligent-Tiering** (automatic, no manual age thresholds — objects move between Frequent/Infrequent/Archive Instant Access tiers based on actual access) |
| `cloudmart-documents` | Predictable: hot for 30 days, cold after | Standard → Standard-IA (30d) → Glacier Flexible Retrieval (90d) |
| `cloudmart-backups` | Rarely read, must survive disaster | Standard → Glacier Deep Archive (7d) directly |
| `cloudmart-logs` | Compliance retention, almost never read | Standard-IA immediately (`PUT` directly to STANDARD_IA where SDK supports it) → Deep Archive (30d) → Expire (2555 days / 7 yrs) |
| `cloudmart-archive` | Legal hold candidates | Glacier Deep Archive + Object Lock Compliance mode |

Also configure, and be ready to explain the cost impact of each:
- Cleanup of incomplete multipart uploads (7 days) — orphaned parts silently accrue storage cost.
- Expiration of noncurrent versions (e.g., 90 days after becoming noncurrent) — versioning without a noncurrent-version lifecycle rule is one of the most common real-world S3 cost leaks.
- S3 Intelligent-Tiering's **Archive Access** and **Deep Archive Access** optional tiers for `images/` if long-tail assets exist (old product photos).

---

## 7. Object Lock & Legal/Compliance Requirements

The original brief says "Object Lock for legal documents." Production-level detail:

- Object Lock **must** be enabled at bucket creation — it cannot be retrofitted. This is a common trap; the capstone should include a deliberate step where you discover this (e.g., try enabling it on an existing bucket, observe the failure, and document the remediation of creating a new bucket + copying data).
- Use **Governance mode** for internal retention policies that an authorized role can override, and **Compliance mode** for regulatory holds (e.g., 7-year financial retention) that *nobody*, including the root user, can shorten or delete before expiry.
- Set a **default retention period** at the bucket level so every new object is automatically protected without relying on the uploader to set it.
- Combine with **Legal Hold** for documents under active litigation — independent of the retention period, removable only by an authorized principal.

**Deliverable:** attempt to delete a locked object before its retention date expires and capture the `AccessDenied` response as evidence for your documentation.

---

## 8. Replication — Cross-Region *and* Cross-Account

Original brief: Cross-Region Replication to a second region. Production-level addition: replicate to a **different AWS account** in the DR region too, so a compromised or deleted account doesn't take out both copies.

Requirements:
- Enable **S3 Replication Time Control (RTC)** for the `finance/` and `archive/` prefixes — gives a 15-minute SLA on replication and CloudWatch metrics (`ReplicationLatency`, `OperationsFailedReplication`) you can alarm on.
- Replicate KMS-encrypted objects (requires the destination bucket's KMS key to be specified in the replication rule, and a policy on the source key allowing the replication role to `kms:Decrypt`).
- Enable **delete marker replication** deliberately as an explicit choice — document why you turned it on or off (turning it on means a delete in the source propagates to DR; turning it off means DR retains everything even if source data is deleted, which is safer against ransomware/accidental deletion but changes your DR bucket's cost profile).
- Replicate to a bucket with **S3 Object Lock enabled** in the DR account for the archive/legal bucket, so replicated legal holds remain immutable even in the failover region.

---

## 9. Access Points & Multi-Region Access Points

Original brief creates department-scoped Access Points. Add:

- A **Multi-Region Access Point (MRAP)** in front of the `cloudmart-images` bucket and its CRR replica, giving applications a single global endpoint that automatically routes to the lowest-latency, available region — relevant if CloudMart's storefront serves multiple geographies.
- **VPC-restricted Access Points** for the finance/HR access points (`vpc` configuration on the access point itself, not just the bucket policy) — this is a second, independent network control, not a duplicate of Section 4.

---

## 10. Monitoring, Auditing & Threat Detection

Original brief: review CloudWatch/CloudTrail/Storage Lens. Production-level engineers are expected to also wire up *detective and preventive* controls, not just dashboards:

| Control | Purpose |
|---|---|
| **AWS Config Rules** (`s3-bucket-public-read-prohibited`, `s3-bucket-server-side-encryption-enabled`, `s3-bucket-versioning-enabled`, `s3-bucket-replication-enabled`) | Continuous compliance — alerts (or auto-remediates via SSM Automation) if a bucket drifts out of policy |
| **GuardDuty S3 Protection** | Detects anomalous API activity against S3 (e.g., unusual `GetObject` volume from an unfamiliar IP, potential credential compromise) |
| **Macie** | Scans `documents/` and `finance/` for PII/PCI data and classifies sensitivity — critical because CloudMart stores customer and financial data |
| **CloudWatch Alarms** on `4xxErrors`, `5xxErrors`, `ReplicationLatency`, and `BucketSizeBytes` anomalies | Operational health, not just security |
| **CloudTrail Data Events** with a **CloudTrail Lake** query or Athena-over-S3-access-logs setup | Ability to answer "who accessed object X, when" during an incident — the original brief enables logging but doesn't require you to *query* it |

**Deliverable (new):** run one simulated incident — e.g., have a test IAM principal without permission attempt `GetObject` on `finance/` — and produce the CloudTrail/Athena query that surfaces the denied attempt, plus the GuardDuty finding if triggered.

---

## 11. Cost Optimization — With Numbers

The original brief asks qualitative questions ("is Intelligent-Tiering appropriate?"). Production-level work quantifies it.

Deliverable: a short cost model, e.g.

```
Bucket: cloudmart-logs
Current: 2 TB in S3 Standard, avg age 400 days, <1% accessed after 30 days
Standard cost/mo:        2,000 GB × $0.023        = $46.00
Proposed (STANDARD_IA immediately + Deep Archive @30d):
  30 days STANDARD_IA:   2,000 GB × $0.0125        = $25.00 (prorated)
  Steady-state Deep Archive: 2,000 GB × $0.00099    = $1.98/mo
  Estimated retrieval requests (rare): negligible
Projected savings: ~95% at steady state after 30-day transition window
```

Also require:
- A **Storage Lens** dashboard screenshot/export showing the top cost-driving prefixes.
- Identification of any bucket where **noncurrent version storage** exceeds current-version storage (a very common, very expensive misconfiguration).
- A recommendation on **S3 Requester Pays** if any bucket serves cross-account consumers who should bear their own retrieval cost.

---

## 12. Disaster Recovery — With RTO/RPO and a Runbook

Original brief: test recovery scenarios informally. Production-level: define targets *before* testing against them.

| Scenario | RTO Target | RPO Target | Mechanism |
|---|---|---|---|
| Accidental object delete | < 5 min | 0 (no data loss) | Versioning + documented restore procedure (`aws s3api list-object-versions` → `copy-object` of prior version) |
| Bucket-level data loss (misconfigured lifecycle expiry, ransomware) | < 1 hr | ≤ 15 min | CRR with RTC + Object Lock on DR copy |
| Full region outage | < 4 hr | ≤ 15 min | Failover to MRAP-routed DR region, DNS/app config cutover documented in runbook |
| Legal/compliance record loss | 0 (must never happen) | 0 | Object Lock Compliance mode, cross-account replication |

**Deliverable:** a written runbook (numbered steps, exact CLI commands, named owners/roles) for at least the first two scenarios, plus evidence (command output) that you executed scenario 1 end-to-end.

---

## Consolidated Task List (Supersedes Original Phases 1–12)

1. Provision multi-account structure (or simulated OU boundaries).
2. Author Terraform module for the 5 buckets + KMS keys; deploy via a documented plan/apply flow.
3. Implement SSO-based roles (HR, Finance, Developers, Marketing, Operations) with condition-scoped, MFA-aware policies; validate with Access Analyzer and Policy Simulator.
4. Apply layered bucket policies: deny public, deny non-HTTPS, deny non-KMS uploads, deny TLS < 1.2, restrict to VPC endpoint.
5. Build folder structure and upload representative sample data (include some PII-like test data for Macie to find).
6. Wire EventBridge-based event routing per the table in Section 5, including DLQs and failure destinations.
7. Configure access-pattern-justified lifecycle rules per bucket (Section 6), plus incomplete-multipart and noncurrent-version cleanup.
8. Enable Object Lock at bucket creation for the archive bucket; set default retention; test Governance vs Compliance mode behavior.
9. Configure CRR + cross-account replication with RTC for critical prefixes; verify with a timed test upload.
10. Create department Access Points plus one MRAP; write distinct policies per access point.
11. Enable Config rules, GuardDuty S3 Protection, Macie, and CloudTrail Data Events with Lake/Athena queryability; run the simulated-incident deliverable.
12. Produce the quantified cost model and Storage Lens findings.
13. Write and execute the DR runbook with RTO/RPO targets; capture evidence.
14. Write a one-page **architecture decision record (ADR)** for each major choice (KMS vs SSE-S3, EventBridge vs direct notifications, Governance vs Compliance mode, delete-marker replication on/off) — this is what separates "I clicked the buttons" from "I can defend the design."

---

## Expanded Self-Assessment Checklist

| Task | Status |
|---|---|
| Multi-account boundary documented | ☐ |
| Terraform module deployed for all buckets | ☐ |
| SSO roles with condition-scoped policies | ☐ |
| Access Analyzer run with zero unintended findings | ☐ |
| KMS CMKs with rotation enabled | ☐ |
| Bucket policies enforce HTTPS, TLS 1.2+, KMS, VPC endpoint | ☐ |
| EventBridge routing with DLQs configured | ☐ |
| Lifecycle rules justified by access-pattern data | ☐ |
| Noncurrent version + incomplete multipart cleanup enabled | ☐ |
| Object Lock enabled at creation, Governance vs Compliance tested | ☐ |
| Cross-region + cross-account replication with RTC verified | ☐ |
| Access Points + MRAP created with distinct policies | ☐ |
| Config rules, GuardDuty S3 Protection, Macie enabled | ☐ |
| Simulated incident traced via CloudTrail/Athena | ☐ |
| Quantified cost model produced | ☐ |
| DR runbook written and executed with evidence | ☐ |
| ADRs written for major design decisions | ☐ |

---

## Production Interview Challenge (Expanded)

Beyond the original questions, be ready to answer:

- Why did you choose customer-managed KMS keys over SSE-S3, and what's the operational cost of that choice (key policies, rotation, cross-account grants)?
- Why route through EventBridge instead of wiring S3 notifications directly to Lambda/SNS/SQS?
- What's the difference between MFA Delete (bucket-level) and MFA-conditioned IAM policies, and why might you choose one over the other?
- Walk through what happens, step by step, if someone tries to delete a Compliance-mode locked object — including as root.
- Your CRR replication latency alarm just fired. What do you check first?
- A Config rule flags a bucket as non-compliant for public access. Walk through your remediation — manual or automated (SSM Automation document)?
- The finance team says a report from 45 days ago is "gone." Walk through your exact recovery steps.
- How would you estimate the monthly S3 bill for a bucket before you've collected any Storage Lens data?
- If CloudMart's storefront now needs single-digit-millisecond image reads for 5 different regions, how does your architecture change?
- What's your redrive strategy for the DLQ on the `logs/` SQS queue?

---

## Beyond This Capstone

Same list as before, now with the connective tissue you should be able to articulate:

- **CloudFront** — cache `images/` at the edge; also lets you drop the S3 bucket policy's public-read requirement entirely by using Origin Access Control (OAC).
- **AWS Backup** — centralized backup policy that can also manage S3 backup/restore alongside EBS/RDS, useful once CloudMart's estate grows beyond S3.
- **Macie / Lake Formation / Glue / Athena / Redshift / EMR** — the analytics and governance layer that consumes what you've built here.
- **EventBridge / Organizations / Control Tower** — the org-wide automation and guardrail layer this capstone assumes but doesn't fully build.
- **Terraform CI/CD** — turn the module from Section 2 into a pipeline with plan-on-PR, apply-on-merge, and drift detection (`terraform plan` on a schedule, alert on non-empty diff).

---

## Final Words

The original capstone teaches you to configure S3 correctly. This production edition asks you to also defend *why* — with cost numbers, RTO/RPO targets, condition-scoped IAM, and an incident you actually traced end to end. That's the difference between "I completed an S3 course" and "I can operate S3 in production," which is the bar a 3–5 year Cloud Engineer interview will actually test.


Sure — let me put my "lead DevOps engineer at CloudMart" hat on and walk you through the reasoning the way I'd actually present it in a design review.

---

**Why multi-account instead of one account with folders/tags?**

The first question I ask on any new platform is: what's my blast radius if something goes wrong? A single account with IAM policies as the only boundary means one bad policy, one leaked access key, or one compromised CI/CD role in "dev" can touch the same finance and legal data that "prod" depends on. IAM is a *logical* boundary — it can be misconfigured. An account boundary is a *hard* boundary enforced by AWS itself. Given that CloudMart stores financial reports and legal documents, I wasn't willing to bet compliance on a policy never having a typo. So dev/staging/prod get their own accounts, and I split out a dedicated log-archive account specifically so that even someone with admin rights in prod can't quietly delete the audit trail of what they just did.

**Why KMS customer-managed keys instead of just SSE-S3?**

SSE-S3 is free and zero-effort, and honestly for the `images/` bucket it'd probably be fine. But for `finance/` and the legal archive, I need two things SSE-S3 can't give me: a key policy I control (so I can restrict *who* can decrypt, not just who can read the object), and an audit trail of every decrypt call via CloudTrail. If we ever get asked "prove that only Finance could read these documents," SSE-S3 gives me nothing to point to. KMS costs a bit more and adds key rotation and cross-account grant complexity for replication — I accepted that operational overhead because the alternative is not being able to answer an audit question.

**Why roles via SSO instead of IAM users?**

Long-lived access keys are the single most common root cause I've seen in incident postmortems — they get committed to repos, they don't rotate, they outlive the employee. Roles assumed through SSO are short-lived by default and tie access back to a real identity provider, so when someone leaves, access disappears the moment their SSO account is deactivated instead of requiring someone to remember to deactivate an IAM user separately.

**Why condition-scoped policies (MFA, TLS version, VPC endpoint) instead of just resource ARNs?**

A resource-scoped policy answers "can this identity touch this object." It doesn't answer "was this a low-risk request." Requiring MFA for deletes, TLS 1.2+ for all requests, and traffic to originate from our VPC endpoint means that even a fully-authorized identity making a request from an unexpected context gets blocked. This is defense in depth — I'm not trusting any single control to be the only thing standing between us and a bad day.

**Why EventBridge instead of wiring S3 notifications straight to Lambda/SNS/SQS?**

Direct S3 event notifications work, but a bucket only supports one notification configuration, and every time a new team wants to consume `images/` upload events, someone has to go edit that config — which means touching infrastructure that other teams already depend on. Routing through EventBridge means new consumers just add a rule on the bus. Nobody has to touch the S3 config again. It's a small extra hop, but it decouples "who owns the bucket" from "who consumes its events," which matters a lot once more than one team is involved.

**Why Intelligent-Tiering for images but manual lifecycle rules for documents/backups?**

Images have unpredictable access patterns — some product photos stay hot for a campaign, most go cold fast, and I don't want to babysit that. Intelligent-Tiering handles it automatically for a small monitoring fee. But documents and backups have *predictable* patterns — a financial report is essentially never read after 90 days — so paying for Intelligent-Tiering's monitoring fee there is wasted money. I'd rather set explicit transitions I can defend with actual access data from Storage Lens than pay for automation I don't need.

**Why Object Lock in Compliance mode for legal documents specifically, not everywhere?**

Compliance mode is intentionally unforgiving — not even the root user can override it before expiry. That's exactly what I want for a document under legal hold, and exactly what I *don't* want for, say, a config backup where an engineer might legitimately need to shorten retention with authorization. So I use Governance mode as the default and reserve Compliance mode for the one bucket where the requirement is genuinely "nobody, ever, no exceptions."

**Why cross-account replication on top of cross-region?**

Cross-region protects me from a regional outage. It does not protect me from someone with legitimate prod credentials deleting or encrypting data maliciously (ransomware-style) — because that same identity often has access to both regions. Putting the DR copy in a separate account with its own, more restrictive access model means a compromised prod credential can't reach the backup at all.

**Why RTC (Replication Time Control) only on `finance/` and `archive/`, not everything?**

RTC costs more per GB replicated. I don't need a 15-minute SLA on product image replication — if it takes two hours, nobody notices. I do need it on the prefixes tied to financial and legal recovery targets, because that's where I've committed to an RTO the business actually cares about. Applying it everywhere would just be spending money to hit an SLA nobody asked for.

**Why GuardDuty + Macie + Config on top of "just enable CloudTrail"?**

CloudTrail tells you what happened *after* you go looking. GuardDuty and Config are the difference between finding out about a problem during an audit six months later versus getting paged within minutes. Macie specifically earns its keep because we're storing customer and financial data — I want to know if PII ends up somewhere it shouldn't (like an `images/` bucket that's more loosely governed) before a customer or regulator finds it for me.

**Why write ADRs for these choices at all?**

Because six months from now, someone — maybe me — is going to ask "why is this so complicated, can we just turn off KMS and use SSE-S3." Without a written record of the trade-off I made and why, that conversation starts from zero every time. The ADRs are how I make sure the next engineer doesn't have to re-derive my reasoning, or worse, undo a control that exists for a compliance reason nobody remembers.

---

The common thread: almost every "extra" piece in this design exists because I asked *what happens when this control fails or is bypassed*, not just *does this satisfy the requirement*. That's usually the question that separates a working demo from something I'd actually be willing to put my name on in production.
