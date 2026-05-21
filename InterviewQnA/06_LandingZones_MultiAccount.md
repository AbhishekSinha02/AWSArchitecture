# AWS Interview Q&A — Landing Zones, Multi-Account Strategy & Governance

> **The Enterprise Story:** A startup runs everything in one AWS account. A year later,
> they have 200 developers, 15 environments, a security breach because a dev had admin access
> to production, and a $200,000 surprise bill because someone spun up GPU instances for
> crypto mining. Landing Zones exist to prevent all of that — to give organizations a
> structured, secure, governed AWS foundation that scales from 10 to 10,000 developers.

---

## Table of Contents
1. [Why Multi-Account?](#why)
2. [AWS Organizations & Account Structure](#orgs)
3. [Service Control Policies (SCPs)](#scps)
4. [AWS Control Tower](#controltower)
5. [Account Factory for Terraform (AFT)](#aft)
6. [Centralized Logging & Security](#logging)
7. [Identity Federation & SSO](#sso)

---

## 1. Why Multi-Account? {#why}

---

### Q1: Why should a company use multiple AWS accounts instead of one?

**The Story:**
In the early days, companies put everything in one account: dev workloads, staging, production,
finance apps, customer data, machine learning experiments — all sharing one namespace, one
billing profile, one security boundary.

Then one day, a developer testing a new autoscaling script accidentally terminated 300 production
EC2 instances because the IAM permissions were too broad. All in the same account.

Multiple accounts solve this with four superpowers:

```
1. BLAST RADIUS ISOLATION:
   If a dev account is compromised or accidentally destroyed,
   production accounts are completely unaffected.
   Network isolation is enforced at the account level — no IAM policy mistakes
   can accidentally bridge prod and dev.

2. BILLING CLARITY:
   Each account has separate billing.
   "How much does the ML team cost?" → filter by account.
   Cost allocation is automatic — no tagging discipline required.

3. SECURITY BOUNDARIES:
   IAM is scoped per account. A dev cannot assume a role in prod
   unless explicitly allowed by a cross-account role trust policy.
   Service Control Policies (SCPs) enforce guardrails per account group.

4. COMPLIANCE SEGMENTATION:
   PCI DSS scope: only payment processing accounts.
   HIPAA scope: only healthcare accounts.
   SOC 2: audit only the relevant account set.
   You're not dragging everything into every compliance scope.
```

**The Rule of Thumb:**
> "When in doubt, give it its own account. Accounts are free. Incidents are expensive."

---

## 2. AWS Organizations & Account Structure {#orgs}

---

### Q2: What is the recommended multi-account structure for a mid-to-large enterprise?

**The Story:**
This is the AWS Landing Zone blueprint that every large company ends up at, eventually.
The smart ones start here.

```
ROOT (Management Account)
  Purpose: ONLY for billing, organization management, and SCP attachment
  Never run workloads here
  ──────────────────────────────────────────────────────────────────────

  ├── INFRASTRUCTURE OU
  │   ├── Network Account          → Transit Gateway, Direct Connect, DNS
  │   ├── Security Tooling Account → Security Hub, GuardDuty Master, Macie, Config aggregator
  │   ├── Log Archive Account      → Central S3 bucket for all CloudTrail + Config logs
  │   └── Shared Services Account  → Active Directory, Internal tools, Monitoring (Grafana)
  │
  ├── WORKLOADS OU
  │   ├── PROD OU
  │   │   ├── app1-prod            → Production workloads, strictest SCPs
  │   │   ├── app2-prod
  │   │   └── data-prod
  │   │
  │   ├── NON-PROD OU
  │   │   ├── app1-dev
  │   │   ├── app1-staging
  │   │   └── data-dev
  │   │
  │   └── SANDBOX OU
  │       └── developer-sandbox-*  → Devs can experiment freely; SCPs prevent prod-like actions
  │                                   Auto-terminated after 30 days
  │
  ├── SECURITY OU
  │   ├── Audit Account            → Cross-account Config Rules, CloudTrail, audit
  │   └── Backup Account           → Centralized backups (AWS Backup)
  │
  └── EXCEPTIONS OU
      └── Legacy Account           → Older accounts being migrated; extra guardrails
```

**The key insight for interviews:** Separation is at the OU level, and SCPs attach at the OU level.
"Production OU" SCPs can deny: resource deletion, disabling security services, region creation.
"Sandbox OU" SCPs can deny: expensive instance types, production data access, cross-account network access.

---

## 3. Service Control Policies (SCPs) {#scps}

---

### Q3: What are SCPs and how do they differ from IAM policies?

**The Story:**
An SCP is not a permission grant — it's a permission ceiling. It defines the maximum possible
permissions any identity in an account (including the account root!) can ever have.

```
IAM POLICY:
  "User Alice is ALLOWED to terminate EC2 instances"
  → Grants permissions to a specific identity

SCP:
  "No identity in this account can EVER terminate EC2 instances in us-west-2"
  → Sets the ceiling, regardless of what any IAM policy says

THE MATH:
  Effective Permissions = IAM Policy ∩ SCP
  (the intersection — only what BOTH allow)

EXAMPLE:
  IAM Policy: Allow { ec2:* } (everything)
  SCP:        Deny  { ec2:TerminateInstances }
  Result:     Can do EVERYTHING except terminate (SCP wins)

  IAM Policy: Allow { s3:PutObject on bucket "finance-data" }
  SCP:        Allow { s3:* } (everything S3)
  Result:     Can only PutObject on finance-data (IAM is more restrictive)
```

---

### Q4: Give me 5 real SCP examples an enterprise would use.

```
SCP 1 — PREVENT REGION CREATION (Data Residency):
{
  "Effect": "Deny",
  "Action": ["account:EnableRegion"],
  "Resource": "*"
}
→ Prevents enabling regions like Bahrain or South Africa for EU-data-residency compliance

SCP 2 — BLOCK DISABLING SECURITY SERVICES:
{
  "Effect": "Deny",
  "Action": [
    "guardduty:DeleteDetector",
    "guardduty:DisassociateFromMasterAccount",
    "cloudtrail:DeleteTrail",
    "cloudtrail:StopLogging",
    "config:DeleteConfigurationRecorder",
    "securityhub:DisableSecurityHub"
  ],
  "Resource": "*"
}
→ Even account admins cannot turn off security monitoring
→ Attacker who compromises the account cannot cover their tracks

SCP 3 — RESTRICT TO APPROVED REGIONS ONLY:
{
  "Effect": "Deny",
  "Action": "*",
  "Resource": "*",
  "Condition": {
    "StringNotEquals": {
      "aws:RequestedRegion": ["us-east-1", "us-west-2", "eu-west-1"]
    }
  }
}
→ All activity must happen in approved regions only (GDPR, data residency)
→ Exempt global services: IAM, Route53, CloudFront, S3 (special handling needed)

SCP 4 — REQUIRE ENCRYPTION (Compliance):
{
  "Effect": "Deny",
  "Action": "s3:PutObject",
  "Resource": "*",
  "Condition": {
    "Null": { "s3:x-amz-server-side-encryption": "true" }
  }
}
→ All S3 PutObject requests must include an encryption header
→ Unencrypted uploads are blocked at the network level, not just policy

SCP 5 — PREVENT EXPENSIVE INSTANCE TYPES (Cost Control in Sandbox):
{
  "Effect": "Deny",
  "Action": "ec2:RunInstances",
  "Resource": "arn:aws:ec2:*:*:instance/*",
  "Condition": {
    "StringLike": {
      "ec2:InstanceType": ["p3.*", "p4d.*", "inf1.*", "g4dn.*"]
    }
  }
}
→ Sandbox OU: developers cannot spin up GPU instances for crypto mining
→ Only applies to sandbox; prod OU has no such restriction (ML team needs GPUs)
```

---

## 4. AWS Control Tower {#controltower}

---

### Q5: What is AWS Control Tower and what does it automate?

**The Story:**
Building a Landing Zone manually is a 6-week project: create the Organizations structure,
configure CloudTrail in every account, set up Security Hub, create the log archive account,
build an account vending machine, write 30 SCPs, configure SSO...

AWS Control Tower does all of that in an afternoon. It's an opinionated, pre-built Landing Zone.

```
WHAT CONTROL TOWER SETS UP FOR YOU:

1. AWS Organizations structure (with recommended OU layout)

2. Pre-configured accounts:
   → Management Account (root)
   → Log Archive Account (centralized CloudTrail + Config logs, S3 with Object Lock)
   → Audit Account (Security tooling, read-only cross-account access)

3. Mandatory Guardrails (you cannot turn these off):
   → CloudTrail enabled in all accounts
   → Config recording enabled in all accounts
   → Log Archive bucket protected from deletion
   → Root user activity alerted

4. Strongly Recommended Guardrails:
   → MFA required for root
   → S3 Block Public Access enabled
   → EBS encryption by default

5. Account Factory:
   → Service Catalog-based self-service account provisioning
   → "I need a new prod account for team X" → fills a form → account created in 15 minutes
   → Accounts come pre-configured with VPC, security tools, baseline SCPs, SSO access

6. AWS IAM Identity Center (SSO):
   → Centralized user access management
   → Developer logs into one portal → gets access to approved accounts
   → No more individual IAM users per account
```

---

### Q6: What are the limitations of Control Tower? When would you NOT use it?

**The Story:**
Control Tower is excellent for starting fresh. But it's opinionated and has constraints.

```
LIMITATIONS:

1. Not designed for brownfield (existing complex Organizations setup)
   → If you have 200 accounts already, retrofitting Control Tower is painful
   → It wants to own the Organizations structure

2. Account Factory customization is limited (without AFT)
   → Default Account Factory can't run custom Terraform or deploy specific resources
   → Use Account Factory for Terraform (AFT) for advanced customization

3. Guardrail coverage is AWS service-specific
   → No coverage for third-party tools or custom policies
   → Custom SCPs must be managed separately

4. Landing Zone updates take time
   → When Control Tower releases updates, you must manually apply the landing zone update
   → Can take 30-60 minutes and briefly restricts account provisioning

WHEN TO USE ALTERNATIVES:
   → Large orgs with complex requirements: build custom Landing Zone with Terraform
   → Orgs already using Terraform everywhere: use the AWS Landing Zone Accelerator (LZA)
     or community terraform-aws-landing-zone module
   → Need full IaC control: Account Factory for Terraform (AFT) extends Control Tower
```

---

## 5. Account Factory for Terraform (AFT) {#aft}

---

### Q7: What is AFT (Account Factory for Terraform)?

**The Story:**
Standard Control Tower Account Factory uses AWS Service Catalog — a point-and-click form to
provision accounts. It works, but it's limited: you can't run custom Terraform after account creation,
you can't parameterize deeply, and it doesn't integrate with GitOps workflows.

AFT is a Terraform-based extension that makes account provisioning entirely Git-driven:

```
AFT WORKFLOW:

Developer creates a PR in Git (account-requests repo):

  account-requests/
    app1-prod-account.tf:
      module "account" {
        source  = "aws-ia/control_tower_account_factory/aws"
        name    = "app1-prod"
        email   = "aws+app1-prod@company.com"
        ou      = "Workloads/Prod"
        sso_email = "platform-team@company.com"
        tags    = { "owner" = "app-team", "env" = "prod" }
      }

PR reviewed → merged → pipeline triggers:

  STEP 1: Control Tower creates the account
  STEP 2: AFT applies "account baseline" Terraform:
          → Deploys: VPC, security tools, IAM roles, logging
  STEP 3: AFT applies "customizations":
          → Team-specific: deploy EKS cluster, create S3 buckets, set budgets
  STEP 4: SSO permission sets assigned
  STEP 5: Slack notification: "Account app1-prod is ready!"

RESULT:
  → New AWS account, fully configured, in 20-30 minutes
  → All account settings are in Git (auditable, repeatable, version-controlled)
  → Add a new security baseline? Update the baseline Terraform → re-run against all accounts
```

---

## 6. Centralized Logging & Security {#logging}

---

### Q8: How do you design a centralized logging architecture across 50+ AWS accounts?

**The Story:**
Each AWS account generates CloudTrail logs, Config logs, VPC flow logs, security findings.
If each account stores them independently, you have 50 siloed log stores. Auditing a cross-account
incident means logging into 50 consoles. That's how attackers cover their tracks.

```
CENTRALIZED LOGGING ARCHITECTURE:

ALL ACCOUNTS:
  CloudTrail → S3 in LOG ARCHIVE ACCOUNT (cross-account delivery)
  VPC Flow Logs → CloudWatch → Kinesis Firehose → S3 in LOG ARCHIVE ACCOUNT
  Config → S3 in LOG ARCHIVE ACCOUNT
  Security Hub → delegated admin in SECURITY TOOLING ACCOUNT
  GuardDuty → delegated admin in SECURITY TOOLING ACCOUNT

LOG ARCHIVE ACCOUNT:
  S3 Bucket (org-central-logs):
    → S3 Object Lock (WORM): logs cannot be deleted for 7 years
    → Bucket policy denies Delete to everyone (including bucket owner)
    → Only PUT allowed from Organization member accounts
    → Replication to a second region (DR for logs)
  
  Query:
    Athena tables on top of S3 → "Show all CloudTrail events in account X last 30 days"
    OpenSearch Service → real-time log search and dashboards (Kibana)

SECURITY TOOLING ACCOUNT:
  AWS Security Hub (aggregator):
    → Receives findings from ALL accounts: GuardDuty, Inspector, Macie, Config
    → Single pane of glass: "Show me all CRITICAL findings across the org"
    → Integration: Slack alerts, Jira tickets, automated remediation via Lambda

AUTOMATION:
  Security Hub finding → EventBridge → Lambda:
    Finding: "S3 bucket is public in account 12345"
    Lambda: applies S3 Block Public Access to that bucket
    → Auto-remediation, not just alerting
```

---

## 7. Identity Federation & SSO {#sso}

---

### Q9: How does AWS IAM Identity Center work and how is it different from traditional IAM?

**The Story:**
Before IAM Identity Center (formerly SSO), managing access for 100 developers across 20 AWS
accounts looked like this: 100 IAM users per account × 20 accounts = 2,000 user records to maintain.
When an employee leaves, you hope someone remembers to delete all 2,000. (They won't.)

IAM Identity Center centralizes this:

```
IAM IDENTITY CENTER ARCHITECTURE:

Identity Source (pick one):
  Option A: AWS IAM Identity Center built-in directory
  Option B: Microsoft Active Directory (via AWS AD Connector or AWS Managed AD)
  Option C: External IdP via SAML 2.0 (Okta, Azure AD, Ping Identity)
  
           ↓ (user authenticates once)
  
IAM Identity Center:
  Permission Sets: define what access a role type gets
    "Developer" → ReadOnlyAccess in prod, AdminAccess in dev
    "DataScientist" → S3FullAccess, SageMaker access
    "SecurityAuditor" → SecurityAudit policy, read-only everywhere
  
  Account Assignments:
    Developer group → Developer permission set → app1-dev, app1-staging accounts
    SecurityAuditor group → SecurityAuditor permission set → ALL accounts
  
           ↓ (user selects account in portal)
  
AWS Account:
  IAM Identity Center generates temporary credentials (AssumeRoleWithSAML)
  User gets a role session for 1-12 hours (configurable)

RESULT:
  - Developer logs into company.awsapps.com/start (one URL)
  - Selects "app1-dev" → gets temporary AWS Console or CLI access
  - CLI: aws configure sso (sets up credential refresh automatically)
  - Zero IAM users to maintain in individual accounts
  - User leaves company: disable in AD/Okta → immediately loses all AWS access
```

---

### Q10: How does cross-account access work with IAM roles?

```
SCENARIO: 
  Security tooling in Account A needs to read CloudTrail logs from Account B

STEP 1: Create a role in Account B (the resource account)
  Role name: "SecurityAuditRole"
  Trust policy: "Account A (account ID 111111111111) can assume this role"
  {
    "Effect": "Allow",
    "Principal": { "AWS": "arn:aws:iam::111111111111:root" },
    "Action": "sts:AssumeRole"
  }
  Permission policy: "ReadOnlyAccess to CloudTrail, S3"

STEP 2: In Account A, give the security tool permission to assume the role
  IAM policy on security tool's role:
  {
    "Effect": "Allow",
    "Action": "sts:AssumeRole",
    "Resource": "arn:aws:iam::222222222222:role/SecurityAuditRole"
  }

STEP 3: Security tool calls STS
  aws sts assume-role \
    --role-arn "arn:aws:iam::222222222222:role/SecurityAuditRole" \
    --role-session-name "SecurityAudit-$(date +%Y%m%d)"
  → Returns temporary credentials (AccessKey, SecretKey, SessionToken)
  → Valid for up to 1 hour
  → Security tool uses these to read Account B's CloudTrail

AT SCALE (org-wide):
  AWS Organizations + CloudFormation StackSets:
  → Deploy the "SecurityAuditRole" to ALL accounts with one click
  → Role exists in every account; trust policy points to security account
  → Security team can audit any account without touching individual IAM
```

---

## Landing Zone Quick Reference

```
CONTROL TOWER GUARDRAIL TYPES:
  Preventive:  SCPs that block non-compliant actions
  Detective:   AWS Config rules that report non-compliance
  Proactive:   CloudFormation hooks that check templates before deploy

ACCOUNT STRUCTURE GOLDEN RULES:
  ✅ Management account: billing and organization management ONLY
  ✅ Log Archive: write-once, cross-account, long retention, never delete
  ✅ Security Tooling: delegated admin for GuardDuty, SecurityHub, Macie
  ✅ Prod accounts: tightest SCPs, no direct developer access (use roles)
  ✅ Sandbox accounts: liberal for experimentation, expire after 30 days

SSO / IDENTITY BEST PRACTICES:
  ✅ Federate with corporate IdP (AD/Okta) — no local IAM users in accounts
  ✅ Permission sets reviewed quarterly — remove unused access
  ✅ Break-glass procedure: emergency IAM user in Log Archive account, MFA required
  ✅ All access is temporary (1-12 hour sessions) — never long-lived keys

COST GOVERNANCE:
  ✅ AWS Budgets on every account (alert at 80% and 100%)
  ✅ Cost Anomaly Detection: ML-based spike detection
  ✅ Tagging policy SCP: deny resource creation without required tags
  ✅ Cost Allocation Tags activated at org root level
```

---

*Next: [07_Migration.md](./07_Migration.md) — The complete guide to AWS migration strategies*
