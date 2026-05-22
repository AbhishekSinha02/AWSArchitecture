# AWS Landing Zones & Governance — How-To Guide

> **Purpose:** Living step-by-step playbook for building, operating, and governing an AWS Landing Zone.
> **Audience:** Senior Architect / Cloud Engineer
> **Style:** Each section leads with the real-world problem, then walks you through the exact steps.
> **Companion to:** [06_LandingZones_MultiAccount.md](./06_LandingZones_MultiAccount.md) — concepts & interview Q&A

---

## How To Use This File

```
STARTING FROM ZERO?     → Sections 1 → 2 → 3 → 4 → 5 in order (foundation first)
EXISTING ACCOUNTS?      → Jump to Section 10 (brownfield onboarding)
SECURITY HARDENING?     → Sections 6 → 7 → 8 → 13
COST GOVERNANCE?        → Section 11
DAILY OPERATIONS?       → Section 14 (health checks) + Section 15 (troubleshooting)
```

---

## Table of Contents

1. [How to Bootstrap AWS Control Tower](#bootstrap)
2. [How to Design Your OU Hierarchy](#ou-design)
3. [How to Write and Apply SCPs Safely](#scps)
4. [How to Set Up Account Factory for Terraform (AFT)](#aft)
5. [How to Provision a New AWS Account End-to-End](#provision)
6. [How to Set Up Org-Wide CloudTrail and Config](#cloudtrail)
7. [How to Deploy GuardDuty Across the Entire Organization](#guardduty)
8. [How to Configure Security Hub as the Central Findings Aggregator](#securityhub)
9. [How to Federate IAM Identity Center with Okta or Azure AD](#sso)
10. [How to Onboard Existing Accounts into Control Tower (Brownfield)](#brownfield)
11. [How to Implement Cost Governance at Scale](#cost)
12. [How to Enforce Tagging Policy Across All Accounts](#tagging)
13. [How to Implement Break-Glass Emergency Access](#breakglass)
14. [How to Run a Landing Zone Health Check](#healthcheck)
15. [Troubleshooting Common Landing Zone Problems](#troubleshooting)

---

## 1. How to Bootstrap AWS Control Tower {#bootstrap}

**The Problem:**
Your company has just decided to adopt AWS at scale. You have a blank management account and
need to build a secure, governed foundation before any workloads go in. Do this wrong and every
account you create will inherit the mistake. Do it right and every future account is born secure.

---

### Pre-Flight Checklist (Do Before Clicking Anything)

```
BEFORE LAUNCHING CONTROL TOWER:

□ CIDR PLANNING
  Decide your VPC CIDR ranges for ALL environments upfront.
  You cannot change CIDRs after VPCs are created.
  
  Recommended allocation:
    Management account VPC:       10.0.0.0/16
    Network/Shared account VPC:   10.1.0.0/16
    Security account VPC:         10.2.0.0/16
    Production accounts:          10.10.0.0/8  (10.10.x.x per account)
    Non-production accounts:      10.20.0.0/8  (10.20.x.x per account)
    Sandbox accounts:             10.30.0.0/8  (10.30.x.x per account)
    On-premises (existing):       192.168.0.0/16  ← MUST NOT overlap above

□ EMAIL ADDRESSES
  You need dedicated email addresses for each vended account.
  AWS account emails cannot be reused.
  Best practice: use an email alias/distribution list, not personal email.
  
  Prepare:
    aws+management@company.com       (root account — already exists)
    aws+log-archive@company.com      (Control Tower creates this)
    aws+audit@company.com            (Control Tower creates this)
    aws+network@company.com          (you'll create this)
    aws+security-tools@company.com   (you'll create this)
    aws+shared-services@company.com  (you'll create this)

□ HOME REGION
  Pick your primary AWS region before launching. This is where Control Tower
  and the Log Archive S3 buckets will live.
  Cannot be changed without re-landing (painful).
  
  Recommendation: us-east-1 (most feature coverage), eu-west-1 (EU compliance)

□ IAM IDENTITY CENTER SOURCE
  Decide BEFORE launching: will you use the built-in Identity Center directory
  or connect your corporate IdP (Okta, Azure AD, Active Directory)?
  
  BUILT-IN:   → Simple, good for < 50 users or greenfield
  OKTA/AZURE: → Recommended for enterprise — connects to corporate HR/directory
  Active Dir: → Use if org already has AWS Managed AD or AD Connector planned

□ ROOT MFA
  Enable MFA on the management account root user IMMEDIATELY.
  Do not proceed to Control Tower setup without this.
```

---

### Step-by-Step: Launch Control Tower

```
STEP 1 — ENABLE CONTROL TOWER
  Console: Management Account → AWS Control Tower → Set up landing zone

STEP 2 — CONFIGURE HOME REGION AND GOVERNED REGIONS
  Home region: us-east-1 (example)
  Additional governed regions: eu-west-1, ap-southeast-1 (if needed)
  
  "Governed" means: Control Tower will deploy guardrails and CloudTrail
  to ALL accounts in these regions. Choose only what you need.
  Cost implication: CloudTrail per-event charges per governed region.

STEP 3 — CONFIGURE FOUNDATIONAL OUs
  Control Tower will create two OUs by default:
    → Security OU  (contains: Log Archive + Audit accounts)
    → Sandbox OU   (for initial experimentation)
  
  Accept defaults or rename to match your naming convention.
  You will add more OUs after setup.

STEP 4 — LOG ARCHIVE ACCOUNT
  Email: aws+log-archive@company.com
  Control Tower creates this account and configures:
    → S3 bucket for org-wide CloudTrail logs
    → S3 Object Lock (WORM — 1 year by default, increase to 7 for compliance)
    → Bucket policy: org accounts can PUT, nobody can DELETE

STEP 5 — AUDIT ACCOUNT
  Email: aws+audit@company.com
  Control Tower creates this account with:
    → Read-only cross-account access to all org accounts
    → SNS topics for Control Tower notifications

STEP 6 — CONFIGURE IAM IDENTITY CENTER
  If using built-in directory: accept default
  If using Okta/Azure AD: select "External identity provider" (configure after launch)
  
  Note: You CANNOT change the identity source after landing zone setup without
  deleting and recreating the landing zone. Decide carefully.

STEP 7 — LAUNCH (takes 30-60 minutes)
  Control Tower creates:
    → 2 accounts (Log Archive, Audit)
    → Organization with Security OU + Sandbox OU
    → 20+ mandatory and optional guardrails as SCPs
    → Org-level CloudTrail
    → AWS Config recording in all accounts
    → IAM Identity Center permission sets

STEP 8 — POST-LAUNCH VERIFICATION
  □ Login to Log Archive account → verify S3 bucket exists with Object Lock
  □ Login to Audit account → verify read-only access to org
  □ Check Control Tower dashboard: all accounts should show "Compliant"
  □ Test: create a user in IAM Identity Center, assign to Sandbox, verify login
```

---

### Post-Bootstrap: Create Additional Foundation Accounts

After Control Tower is running, immediately create these accounts (before any workloads):

```
Account: Network
  OU: Infrastructure
  Email: aws+network@company.com
  Purpose: Transit Gateway, Direct Connect, DNS (Route 53 Resolver)
  
Account: Security Tooling
  OU: Infrastructure
  Email: aws+security-tools@company.com
  Purpose: GuardDuty delegated admin, Security Hub aggregator, Macie
  
Account: Shared Services
  OU: Infrastructure
  Email: aws+shared@company.com
  Purpose: Active Directory, internal tools, shared monitoring (Grafana)

Account: Backup
  OU: Security
  Email: aws+backup@company.com
  Purpose: AWS Backup centralized vault (cross-account backup target)
```

---

## 2. How to Design Your OU Hierarchy {#ou-design}

**The Problem:**
SCPs inherit down the OU tree. Get the structure wrong and you'll either block teams from
doing legitimate work (too restrictive at the wrong level) or fail to enforce guardrails
where you need them (too permissive). Design the tree before you create it.

---

### Decision Tree: What Goes in Which OU?

```
IS THIS ACCOUNT RUNNING PRODUCTION WORKLOADS?
  Yes → WORKLOADS/PROD OU
        Tightest SCPs: no resource deletion, no security service modification
        No direct developer console access (read-only or role-based only)

  No, is it Dev/Staging/QA?
    Yes → WORKLOADS/NON-PROD OU
          SCPs: restrict expensive instance types, no cross-account prod access
          Developers get AdministratorAccess in their own dev accounts

  No, is it for experiments only (no customer data, time-bounded)?
    Yes → SANDBOX OU
          Liberal SCPs: just prevent expensive instances and prod network access
          Auto-expire account after 30 days
          No persistent data allowed

IS THIS ACCOUNT INFRASTRUCTURE/PLATFORM?
  Network, DNS, Transit Gateway? → INFRASTRUCTURE OU
  Security tooling, audit, backup? → SECURITY OU
  Shared services (AD, monitoring)? → INFRASTRUCTURE OU

IS THIS ACCOUNT LEGACY OR MIGRATING?
  → EXCEPTIONS OU
    Extra guardrails to limit blast radius during migration
    Specific SCPs to prevent it reaching prod accounts
```

---

### Reference OU Blueprint

```
ROOT (Management Account — billing + org management ONLY)
│
├── INFRASTRUCTURE OU
│   ├── network-prod              → TGW, DX, Route 53 Resolver
│   ├── shared-services-prod      → AD, Grafana, internal tooling
│   └── backup-prod               → AWS Backup central vault
│
├── SECURITY OU  (Control Tower manages this)
│   ├── log-archive               → Immutable centralized logs
│   └── audit                     → Read-only cross-account, Config aggregator
│
├── SECURITY-TOOLING OU           ← separate from CT Security OU
│   └── security-tools-prod       → GuardDuty admin, Security Hub, Macie
│
├── WORKLOADS OU
│   ├── PROD OU
│   │   ├── app1-prod
│   │   ├── app2-prod
│   │   ├── data-prod             → Data platform, Redshift, S3 data lake
│   │   └── payments-prod         → PCI DSS scope (isolated for compliance)
│   │
│   ├── NON-PROD OU
│   │   ├── app1-dev
│   │   ├── app1-staging
│   │   ├── app2-dev
│   │   └── data-dev
│   │
│   └── SANDBOX OU
│       ├── sandbox-team-platform  → expires 30 days, auto-terminated
│       ├── sandbox-team-ml
│       └── sandbox-personal-*
│
├── POLICY-EXCEPTIONS OU
│   └── legacy-app-migration      → Limited network access, extra audit logging
│
└── SUSPENDED OU
    └── (accounts pending decommission — no SCPs, just isolation)
```

---

### OU Creation Commands (AWS CLI)

```bash
# Get the root ID
ROOT_ID=$(aws organizations list-roots \
  --query 'Roots[0].Id' --output text)

# Create top-level OUs
aws organizations create-organizational-unit \
  --parent-id $ROOT_ID \
  --name "Infrastructure"

aws organizations create-organizational-unit \
  --parent-id $ROOT_ID \
  --name "Security-Tooling"

aws organizations create-organizational-unit \
  --parent-id $ROOT_ID \
  --name "Workloads"

# Create nested OUs under Workloads
WORKLOADS_OU_ID=$(aws organizations list-children \
  --parent-id $ROOT_ID \
  --child-type ORGANIZATIONAL_UNIT \
  --query "Children[?Name=='Workloads'].Id" \
  --output text)

aws organizations create-organizational-unit \
  --parent-id $WORKLOADS_OU_ID \
  --name "Prod"

aws organizations create-organizational-unit \
  --parent-id $WORKLOADS_OU_ID \
  --name "NonProd"

aws organizations create-organizational-unit \
  --parent-id $WORKLOADS_OU_ID \
  --name "Sandbox"
```

---

## 3. How to Write and Apply SCPs Safely {#scps}

**The Problem:**
An SCP applied to the wrong OU, or written with a syntax error, can lock every developer out
of the services they need — in production. A bad SCP is invisible until someone tries to use
the blocked action. Test every SCP before attaching it.

---

### The SCP Safe Deployment Process

```
NEVER attach an untested SCP to a production OU.
Follow this sequence every time:

  STEP 1: Write the SCP (use the templates below)
  STEP 2: Validate JSON syntax (aws organizations describe-policy is not enough)
  STEP 3: Test on a SANDBOX account first (attach to one account, not an OU)
  STEP 4: Verify the sandbox account can still do expected operations
  STEP 5: Verify the blocked action IS blocked (actually test the deny)
  STEP 6: Attach to NON-PROD OU — observe for 48 hours
  STEP 7: Attach to PROD OU only after confirmed working

ROLLBACK:
  aws organizations detach-policy \
    --policy-id p-xxxxxxxxxx \
    --target-id ou-xxxx-xxxxxxxx
  (Takes effect immediately — no propagation delay)
```

---

### How to Create and Attach an SCP

```bash
# STEP 1: Create the policy file
cat > deny-root-actions.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyRootUserActions",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "aws:PrincipalArn": ["arn:aws:iam::*:root"]
        }
      }
    }
  ]
}
EOF

# STEP 2: Create the SCP in the organization
POLICY_ID=$(aws organizations create-policy \
  --content file://deny-root-actions.json \
  --description "Deny all root user actions in non-management accounts" \
  --name "DenyRootUserActions" \
  --type SERVICE_CONTROL_POLICY \
  --query 'Policy.PolicySummary.Id' \
  --output text)

echo "Created SCP: $POLICY_ID"

# STEP 3: Attach to a SINGLE sandbox account first (NOT an OU yet)
SANDBOX_ACCOUNT_ID="123456789012"
aws organizations attach-policy \
  --policy-id $POLICY_ID \
  --target-id $SANDBOX_ACCOUNT_ID

# STEP 4: Test from the sandbox account that root actions are blocked
# (login to sandbox as root, try an action — should be DENIED)

# STEP 5: After validation, attach to the Sandbox OU
SANDBOX_OU_ID="ou-xxxx-xxxxxxxx"
aws organizations attach-policy \
  --policy-id $POLICY_ID \
  --target-id $SANDBOX_OU_ID

# Detach from the single account (now covered by OU)
aws organizations detach-policy \
  --policy-id $POLICY_ID \
  --target-id $SANDBOX_ACCOUNT_ID
```

---

### Production-Ready SCP Library

```json
// SCP-1: DENY DISABLE SECURITY SERVICES
// Attach to: ALL OUs (including Prod and NonProd)
// Purpose: Attacker cannot cover tracks by disabling GuardDuty or CloudTrail
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyDisableSecurityServices",
      "Effect": "Deny",
      "Action": [
        "guardduty:DeleteDetector",
        "guardduty:DisassociateFromAdministratorAccount",
        "guardduty:DisassociateMembers",
        "cloudtrail:DeleteTrail",
        "cloudtrail:StopLogging",
        "cloudtrail:UpdateTrail",
        "config:DeleteConfigurationRecorder",
        "config:DeleteDeliveryChannel",
        "config:StopConfigurationRecorder",
        "securityhub:DisableSecurityHub",
        "securityhub:DeleteMembers",
        "securityhub:DisassociateMembers",
        "macie2:DisableMacie",
        "access-analyzer:DeleteAnalyzer"
      ],
      "Resource": "*"
    }
  ]
}
```

```json
// SCP-2: DENY ACTIONS OUTSIDE APPROVED REGIONS
// Attach to: Workloads/Prod OU, Workloads/NonProd OU
// Exceptions: global services (IAM, STS, Route53, CloudFront, WAF, ACM)
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnapprovedRegions",
      "Effect": "Deny",
      "NotAction": [
        "iam:*", "sts:*", "route53:*",
        "cloudfront:*", "waf:*", "wafv2:*",
        "acm:*", "budgets:*", "ce:*",
        "support:*", "health:*", "account:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": [
            "us-east-1", "us-west-2", "eu-west-1"
          ]
        }
      }
    }
  ]
}
```

```json
// SCP-3: REQUIRE MFA FOR SENSITIVE ACTIONS IN PROD
// Attach to: Workloads/Prod OU
// Purpose: Even if role is compromised, sensitive actions require MFA session
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyHighRiskActionsWithoutMFA",
      "Effect": "Deny",
      "Action": [
        "ec2:TerminateInstances",
        "rds:DeleteDBInstance",
        "rds:DeleteDBCluster",
        "s3:DeleteBucket",
        "dynamodb:DeleteTable",
        "lambda:DeleteFunction",
        "cloudformation:DeleteStack"
      ],
      "Resource": "*",
      "Condition": {
        "BoolIfExists": {
          "aws:MultiFactorAuthPresent": "false"
        }
      }
    }
  ]
}
```

```json
// SCP-4: SANDBOX COST CONTROL
// Attach to: Sandbox OU ONLY
// Purpose: No expensive GPU/memory-optimized instances, no prod-sized resources
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyExpensiveInstances",
      "Effect": "Deny",
      "Action": ["ec2:RunInstances"],
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "StringLike": {
          "ec2:InstanceType": [
            "p3.*", "p4d.*", "p4de.*",
            "inf1.*", "inf2.*",
            "g4dn.*", "g5.*",
            "x1.*", "x2.*",
            "u-*"
          ]
        }
      }
    },
    {
      "Sid": "DenySandboxRDSMultiAZ",
      "Effect": "Deny",
      "Action": ["rds:CreateDBInstance"],
      "Resource": "*",
      "Condition": {
        "Bool": { "rds:MultiAz": "true" }
      }
    }
  ]
}
```

```json
// SCP-5: ENFORCE ENCRYPTION EVERYWHERE
// Attach to: ALL OUs
// Purpose: No unencrypted EBS, no unencrypted S3 uploads
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnencryptedEBSCreate",
      "Effect": "Deny",
      "Action": ["ec2:CreateVolume"],
      "Resource": "*",
      "Condition": {
        "Bool": { "ec2:Encrypted": "false" }
      }
    },
    {
      "Sid": "DenyUnencryptedS3Upload",
      "Effect": "Deny",
      "Action": ["s3:PutObject"],
      "Resource": "*",
      "Condition": {
        "Null": {
          "s3:x-amz-server-side-encryption": "true"
        }
      }
    }
  ]
}
```

---

## 4. How to Set Up Account Factory for Terraform (AFT) {#aft}

**The Problem:**
The default Control Tower Account Factory (via Service Catalog) is click-driven, hard to
audit, and can't run custom Terraform after account creation. AFT makes every account
request a Git pull request — auditable, repeatable, and GitOps-native.

---

### AFT Repository Structure

```
You'll manage FOUR Git repos:

aft-bootstrap/                    ← Deploy AFT infrastructure itself
  main.tf                           (CodePipeline, CodeBuild, Step Functions)

aft-account-requests/             ← Each file = one account to provision
  app1-prod.tf
  app2-prod.tf
  sandbox-ml-team.tf

aft-account-provisioning-framework/  ← Account baseline (applies to ALL accounts)
  terraform/
    cloudtrail.tf
    iam-roles.tf
    vpc.tf
    security-tools.tf
    budgets.tf

aft-account-customizations/       ← Team-specific add-ons (per-account or per-OU)
  app1-prod/
    terraform/
      eks-cluster.tf
      rds-aurora.tf
  sandbox/
    terraform/
      budget-low-limit.tf
```

---

### Step-by-Step: Deploy AFT

```
PREREQUISITES:
  □ Control Tower is running with at least one OU configured
  □ You have a Git provider (GitHub, GitLab, or CodeCommit)
  □ Terraform >= 1.5.0 installed locally
  □ AWS CLI configured with management account admin credentials

STEP 1 — DEPLOY THE AFT BOOTSTRAP MODULE
  Create aft-bootstrap/main.tf:

  module "aft" {
    source = "github.com/aws-ia/terraform-aws-control_tower_account_factory"
    
    # Management account info
    ct_management_account_id = "111111111111"
    log_archive_account_id   = "222222222222"
    audit_account_id         = "333333333333"
    
    # Where AFT pipeline will run (your dedicated AFT account or management)
    aft_management_account_id = "444444444444"
    
    # Git provider for your four repos
    vcs_provider                                  = "github"
    account_request_repo_name                     = "my-org/aft-account-requests"
    account_provisioning_customizations_repo_name = "my-org/aft-account-provisioning-framework"
    account_customizations_repo_name              = "my-org/aft-account-customizations"
    account_request_repo_branch                   = "main"
    
    # Feature flags
    aft_feature_cloudtrail_data_events     = true
    aft_feature_enterprise_support         = false   # set true if you have Enterprise Support
    aft_feature_delete_default_vpcs_enabled = true   # removes default VPC from all new accounts
    
    terraform_version      = "1.5.7"
    terraform_distribution = "oss"  # or "tfe" for Terraform Enterprise
  }

  Run: terraform init && terraform apply
  → AFT deploys: CodePipeline, Step Functions, DynamoDB tables, Lambda functions
  → Takes 15-20 minutes

STEP 2 — CREATE YOUR FIRST ACCOUNT REQUEST
  Create aft-account-requests/app1-prod.tf:

  module "app1_prod" {
    source = "./modules/aft-account-request"
    
    control_tower_parameters = {
      AccountEmail = "aws+app1-prod@company.com"
      AccountName  = "app1-prod"
      ManagedOrganizationalUnit = "Workloads/Prod"
      SSOUserEmail     = "platform-team@company.com"
      SSOUserFirstName = "Platform"
      SSOUserLastName  = "Team"
    }
    
    account_tags = {
      "owner"       = "app-team"
      "environment" = "production"
      "cost-center" = "CC-1234"
      "app"         = "app1"
    }
    
    change_management_parameters = {
      change_requested_by = "platform-engineering"
      change_reason       = "New production account for App1 microservices"
    }
    
    account_customizations_name = "app1-prod"   # maps to aft-account-customizations/app1-prod/
    
    custom_fields = {
      vpc_cidr          = "10.11.0.0/16"
      enable_eks        = "true"
      budget_limit_usd  = "5000"
    }
  }

  git add . && git commit -m "Add app1-prod account request" && git push
  → Pipeline auto-triggers → account provisioned in 20-30 minutes

STEP 3 — VERIFY THE ACCOUNT
  Check CodePipeline in AFT management account → should show green
  Login to new account via IAM Identity Center portal → verify landing page
  Check: VPC exists, CloudTrail active, Config recording, default VPC deleted
```

---

## 5. How to Provision a New AWS Account End-to-End {#provision}

**The Decision Tree: Which path to use?**

```
DO YOU HAVE AFT SET UP?
  YES → Use AFT (Git PR → merge → automated provisioning)
         Step sequence: Create .tf file → PR review → merge → done
  
  NO, do you have Control Tower?
    YES → Use Account Factory (Service Catalog)
           Step sequence: Console → Account Factory → fill form → create
    
    NO → Use AWS CLI directly
           Step sequence: aws organizations create-account → manual baseline

TIMELINE COMPARISON:
  AFT:             20-30 min (fully automated, all baseline included)
  Account Factory: 15-20 min (account created, manual baseline needed)
  CLI direct:      5 min (bare account, no baseline at all)
```

---

### Path A: AFT Account Request (Recommended)

```bash
# 1. Create the account request file
cat > aft-account-requests/sandbox-ml-team.tf << 'EOF'
module "sandbox_ml" {
  source = "./modules/aft-account-request"
  
  control_tower_parameters = {
    AccountEmail              = "aws+sandbox-ml@company.com"
    AccountName               = "sandbox-ml-team"
    ManagedOrganizationalUnit = "Workloads/Sandbox"
    SSOUserEmail              = "ml-lead@company.com"
    SSOUserFirstName          = "ML"
    SSOUserLastName           = "Team"
  }
  
  account_tags = {
    "owner"       = "ml-team"
    "environment" = "sandbox"
    "expiry-date" = "2026-06-30"   # auto-terminated at this date by your lifecycle Lambda
    "cost-center" = "CC-9876"
  }
  
  account_customizations_name = "sandbox"   # uses the sandbox customization profile
  
  custom_fields = {
    budget_limit_usd = "500"      # sandbox budget cap
    enable_sagemaker = "true"
  }
}
EOF

# 2. Commit and push — pipeline starts automatically
git add sandbox-ml-team.tf
git commit -m "Add sandbox account for ML team (expires 2026-06-30)"
git push origin main

# 3. Monitor pipeline
aws codepipeline get-pipeline-state \
  --name aft-account-provisioning-pipeline \
  --query 'stageStates[*].{Stage:stageName,Status:latestExecution.status}' \
  --output table
```

---

### Path B: Account Factory via Console (No AFT)

```
STEP 1: Console → Control Tower → Account Factory → Create Account

STEP 2: Fill in:
  Account name:    sandbox-ml-team
  Account email:   aws+sandbox-ml@company.com
  OU:              Workloads/Sandbox
  SSO email:       ml-lead@company.com
  SSO first name:  ML
  SSO last name:   Team

STEP 3: Wait 15-20 minutes

STEP 4: POST-CREATION BASELINE (manual — this is why AFT is better)
  Login to new account with admin access
  Run your baseline Terraform or CloudFormation StackSet manually:
    □ Delete default VPC (security best practice)
    □ Enable CloudWatch log retention policies
    □ Create standard IAM roles (developer, read-only, etc.)
    □ Set up AWS Budgets alert
    □ Configure VPC with your standard 3-tier layout
```

---

## 6. How to Set Up Org-Wide CloudTrail and Config {#cloudtrail}

**The Problem:**
Each account's CloudTrail logs in its own S3 bucket by default. When a security incident spans
5 accounts, you're logging into 5 different buckets, 5 different Athena queries, 5 Macie setups.
Centralize these on day one — retrofitting later is painful.

---

### Org-Level CloudTrail (Single Trail for All Accounts)

```bash
# Run this from the MANAGEMENT ACCOUNT
# Creates one trail that captures events from ALL org accounts → one S3 bucket

# STEP 1: Create the S3 bucket in LOG ARCHIVE account
# (This should already exist if Control Tower is running — skip if so)

# STEP 2: Update bucket policy to allow org CloudTrail writes
BUCKET_NAME="org-cloudtrail-logs-$(aws sts get-caller-identity --query Account --output text)"
ORG_ID=$(aws organizations describe-organization --query 'Organization.Id' --output text)

cat > bucket-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AWSCloudTrailAclCheck",
      "Effect": "Allow",
      "Principal": {"Service": "cloudtrail.amazonaws.com"},
      "Action": "s3:GetBucketAcl",
      "Resource": "arn:aws:s3:::${BUCKET_NAME}",
      "Condition": {
        "StringEquals": {"aws:SourceOrgID": "${ORG_ID}"}
      }
    },
    {
      "Sid": "AWSCloudTrailWrite",
      "Effect": "Allow",
      "Principal": {"Service": "cloudtrail.amazonaws.com"},
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::${BUCKET_NAME}/AWSLogs/${ORG_ID}/*",
      "Condition": {
        "StringEquals": {
          "s3:x-amz-acl": "bucket-owner-full-control",
          "aws:SourceOrgID": "${ORG_ID}"
        }
      }
    },
    {
      "Sid": "DenyDelete",
      "Effect": "Deny",
      "Principal": "*",
      "Action": ["s3:DeleteObject", "s3:DeleteObjectVersion", "s3:DeleteBucket"],
      "Resource": [
        "arn:aws:s3:::${BUCKET_NAME}",
        "arn:aws:s3:::${BUCKET_NAME}/*"
      ]
    }
  ]
}
EOF

aws s3api put-bucket-policy \
  --bucket $BUCKET_NAME \
  --policy file://bucket-policy.json \
  --profile log-archive-account

# STEP 3: Create the org-level trail (from management account)
aws cloudtrail create-trail \
  --name org-management-events \
  --s3-bucket-name $BUCKET_NAME \
  --is-multi-region-trail \
  --enable-log-file-validation \
  --is-organization-trail \   # ← This is the key flag for org-wide
  --include-global-service-events

# STEP 4: Enable data events (S3 + Lambda) — optional but recommended for compliance
aws cloudtrail put-event-selectors \
  --trail-name org-management-events \
  --event-selectors '[
    {
      "ReadWriteType": "WriteOnly",
      "IncludeManagementEvents": true,
      "DataResources": [
        {"Type": "AWS::S3::Object", "Values": ["arn:aws:s3:::"]},
        {"Type": "AWS::Lambda::Function", "Values": ["arn:aws:lambda"]}
      ]
    }
  ]'

# STEP 5: Start logging
aws cloudtrail start-logging --name org-management-events
```

---

### Org-Level AWS Config (All Accounts, All Regions)

```
HOW TO DEPLOY VIA CLOUDFORMATION STACKSETS:
  StackSets deploy Config recorders to every account in every region automatically.

  Delegated Admin Setup:
    Management account → Organizations → Services → Delegated admin for Config
    → Designate the Audit account as Config delegated administrator

  Conformance Pack — Deploy CIS AWS Benchmark:
    aws configservice put-organization-conformance-pack \
      --organization-conformance-pack-name CIS-Level-2 \
      --template-s3-uri s3://aws-conformance-packs-templates/CIS-Benchmark-Level-2.yaml \
      --delivery-s3-bucket org-config-delivery-bucket \
      --excluded-accounts management-account-id

  What this enforces across ALL accounts automatically:
    → CloudTrail is enabled (rule: cloud-trail-enabled)
    → Root MFA is enabled (rule: root-account-mfa-enabled)
    → EBS volumes are encrypted (rule: encrypted-volumes)
    → S3 Block Public Access enabled (rule: s3-bucket-level-public-access-prohibited)
    → RDS instances are encrypted (rule: rds-storage-encrypted)
    → Security groups don't allow 0.0.0.0/0 on SSH (rule: restricted-ssh)

  Auto-Remediation (for non-compliant resources):
    Config Rule: s3-bucket-level-public-access-prohibited
    Remediation: SSM Automation → AWS-ConfigureS3BucketPublicAccessBlock
    → Non-compliant bucket found → automatically remediated (or raises finding for review)
```

---

## 7. How to Deploy GuardDuty Across the Organization {#guardduty}

**The Problem:**
GuardDuty must be enabled in every account, every region. Without it, an attacker in an
unenabled account has no threat detection. With 50 accounts and 5 regions, that's 250 places
to enable it. Do this with delegated admin — one command, organization-wide coverage.

---

### Step-by-Step: Org-Wide GuardDuty

```bash
# All commands run from MANAGEMENT ACCOUNT

# STEP 1: Designate a delegated administrator (Security Tooling account)
SECURITY_ACCOUNT_ID="555555555555"

aws guardduty enable-organization-admin-account \
  --admin-account-id $SECURITY_ACCOUNT_ID

# STEP 2: Switch to SECURITY TOOLING ACCOUNT for remaining steps

# STEP 3: Enable GuardDuty detector in the security account
DETECTOR_ID=$(aws guardduty create-detector \
  --enable \
  --finding-publishing-frequency FIFTEEN_MINUTES \
  --features '[
    {"Name": "S3_DATA_EVENTS", "Status": "ENABLED"},
    {"Name": "EKS_AUDIT_LOGS", "Status": "ENABLED"},
    {"Name": "EBS_MALWARE_PROTECTION", "Status": "ENABLED"},
    {"Name": "RDS_LOGIN_EVENTS", "Status": "ENABLED"},
    {"Name": "LAMBDA_NETWORK_LOGS", "Status": "ENABLED"}
  ]' \
  --query 'DetectorId' --output text)

echo "Detector ID: $DETECTOR_ID"

# STEP 4: Configure auto-enable for ALL current and future accounts
aws guardduty update-organization-configuration \
  --detector-id $DETECTOR_ID \
  --auto-enable-organization-members ALL \
  --features '[
    {"Name": "S3_DATA_EVENTS",    "AutoEnable": "ALL"},
    {"Name": "EKS_AUDIT_LOGS",   "AutoEnable": "ALL"},
    {"Name": "EBS_MALWARE_PROTECTION", "AutoEnable": "ALL"},
    {"Name": "RDS_LOGIN_EVENTS", "AutoEnable": "ALL"},
    {"Name": "LAMBDA_NETWORK_LOGS", "AutoEnable": "ALL"}
  ]'

# STEP 5: Verify all member accounts are enrolled
aws guardduty list-members \
  --detector-id $DETECTOR_ID \
  --query 'Members[?RelationshipStatus!=`Enabled`].{Account:AccountId,Status:RelationshipStatus}' \
  --output table
# Any account not showing "Enabled" needs manual intervention

# STEP 6: Create SNS → Slack alert for HIGH/CRITICAL findings
# EventBridge Rule in Security Tooling account:
cat > guardduty-alert-rule.json << 'EOF'
{
  "source": ["aws.guardduty"],
  "detail-type": ["GuardDuty Finding"],
  "detail": {
    "severity": [{"numeric": [">=", 7.0]}]
  }
}
EOF

aws events put-rule \
  --name guardduty-high-severity-findings \
  --event-pattern file://guardduty-alert-rule.json \
  --state ENABLED

# → Target: Lambda → formats finding → posts to Slack/PagerDuty
```

---

## 8. How to Configure Security Hub as Central Aggregator {#securityhub}

**The Aggregation Story:**
GuardDuty findings live in the GuardDuty console. Macie findings live in the Macie console.
Inspector findings live in Inspector. Without Security Hub, you're playing whack-a-mole across
12 different consoles. Security Hub pulls them all into one place — one dashboard, one API,
one EventBridge event stream, one Slack channel.

---

### Step-by-Step: Security Hub with Delegated Admin

```bash
# STEP 1: Designate delegated admin (from management account)
aws securityhub enable-organization-admin-account \
  --admin-account-id $SECURITY_ACCOUNT_ID

# STEP 2: From SECURITY TOOLING ACCOUNT — enable Security Hub
aws securityhub enable-security-hub \
  --enable-default-standards \   # CIS AWS Benchmark + AWS Foundational Security Best Practices
  --control-finding-generator SECURITY_CONTROL

# STEP 3: Configure auto-enable for all org accounts
aws securityhub update-organization-configuration \
  --auto-enable \
  --auto-enable-standards DEFAULT

# STEP 4: Enable cross-region aggregation (findings from ALL regions → home region)
aws securityhub create-finding-aggregator \
  --region-linking-mode ALL_REGIONS
# Now ALL findings from ALL regions aggregate into the home region
# One query shows findings from every region across every account

# STEP 5: Enable specific security standards
aws securityhub batch-enable-standards \
  --standards-subscription-requests \
    StandardsArn=arn:aws:securityhub:us-east-1::standards/cis-aws-foundations-benchmark/v/1.4.0 \
    StandardsArn=arn:aws:securityhub:us-east-1::standards/aws-foundational-security-best-practices/v/1.0.0 \
    StandardsArn=arn:aws:securityhub:us-east-1::standards/pci-dss/v/3.2.1

# STEP 6: EventBridge → Lambda → Slack for CRITICAL findings
cat > securityhub-critical-rule.json << 'EOF'
{
  "source": ["aws.securityhub"],
  "detail-type": ["Security Hub Findings - Imported"],
  "detail": {
    "findings": {
      "Severity": {
        "Label": ["CRITICAL", "HIGH"]
      },
      "Workflow": {
        "Status": ["NEW"]
      },
      "RecordState": ["ACTIVE"]
    }
  }
}
EOF

# STEP 7: Auto-remediation for common findings
# Example: auto-close S3 public access findings by fixing the resource
cat > remediation-lambda.py << 'EOF'
import boto3
import json

def handler(event, context):
    finding = event['detail']['findings'][0]
    resource_type = finding['Resources'][0]['Type']
    account_id = finding['AwsAccountId']
    
    if resource_type == 'AwsS3Bucket':
        bucket_name = finding['Resources'][0]['Id'].split(':')[-1]
        region = finding['Resources'][0]['Region']
        
        # Assume cross-account role into the affected account
        sts = boto3.client('sts')
        role = sts.assume_role(
            RoleArn=f'arn:aws:iam::{account_id}:role/SecurityRemediationRole',
            RoleSessionName='SecurityHubRemediation'
        )
        creds = role['Credentials']
        
        s3 = boto3.client('s3',
            aws_access_key_id=creds['AccessKeyId'],
            aws_secret_access_key=creds['SecretAccessKey'],
            aws_session_token=creds['SessionToken'])
        
        # Fix: enable Block Public Access
        s3.put_public_access_block(
            Bucket=bucket_name,
            PublicAccessBlockConfiguration={
                'BlockPublicAcls': True,
                'IgnorePublicAcls': True,
                'BlockPublicPolicy': True,
                'RestrictPublicBuckets': True
            }
        )
        print(f"Remediated: {bucket_name} in account {account_id}")
EOF
```

---

## 9. How to Federate IAM Identity Center with Okta or Azure AD {#sso}

**The Problem:**
When a developer joins your company, IT creates them an account in Okta. When they leave,
IT disables it. Without federation, you'd also need to manually create/delete IAM Identity
Center users. With federation, Okta IS the source of truth — no sync lag, no forgotten accounts.

---

### Path A: Okta Federation

```
STEP 1 — CONFIGURE OKTA APPLICATION
  Okta Admin Console → Applications → Browse App Catalog
  Search: "AWS IAM Identity Center" (official Okta app)
  Add Integration
  
  Configure SAML settings:
    Sign-on URL: from IAM Identity Center console → Settings → SAML metadata
    Audience URI: from IAM Identity Center console → same location
  
  Download: Okta metadata XML file

STEP 2 — CONFIGURE IAM IDENTITY CENTER
  AWS Console → IAM Identity Center → Settings → Identity Source
  → Change identity source → External identity provider
  → Upload the Okta metadata XML
  → Copy the IAM Identity Center ACS URL and Entity ID back to Okta

STEP 3 — CONFIGURE SCIM (AUTOMATIC USER PROVISIONING)
  Without SCIM: users log in via Okta but must exist in Identity Center manually.
  With SCIM: Okta pushes user/group changes to Identity Center automatically.
  
  Identity Center → Settings → Provisioning → Enable automatic provisioning
  → Copy: SCIM endpoint URL + Access token
  
  Okta → AWS IAM Identity Center app → Provisioning tab
  → Enable API integration
  → Paste SCIM endpoint + token
  → Enable: Push Users, Push Groups
  
  Now: Okta group "AWS-Production-Developers" → auto-synced to Identity Center
       Developer added to Okta group → access provisioned in 2 minutes
       Developer removed from Okta → access revoked immediately

STEP 4 — CREATE PERMISSION SETS
  Identity Center → Permission sets → Create:
  
  "Prod-ReadOnly":
    Session duration: 1 hour
    Policies: ReadOnlyAccess (AWS managed)
    Assigned to: Workloads/Prod accounts
    Okta group: AWS-Prod-ReadOnly-Engineers
  
  "Dev-Administrator":
    Session duration: 4 hours
    Policies: AdministratorAccess
    Assigned to: Workloads/NonProd accounts
    Okta group: AWS-Dev-Engineers
  
  "Platform-Administrator":
    Session duration: 2 hours
    Policies: AdministratorAccess
    Assigned to: ALL accounts
    Okta group: AWS-Platform-Engineers
    MFA required: Yes (configure in permission set inline policy)

STEP 5 — VERIFY END-TO-END
  Login: developer goes to https://company.awsapps.com/start
  They see only accounts they have access to
  Selects "app1-dev" → AdministratorAccess → AWS Console opens
  CLI: aws configure sso --profile app1-dev (one-time setup)
       aws s3 ls --profile app1-dev (uses refreshed token automatically)
```

---

### Path B: Azure AD (Entra ID) Federation

```
STEP 1 — AZURE AD ENTERPRISE APP
  Azure Portal → Enterprise Applications → New Application
  Search: "AWS IAM Identity Center"
  Add → configure single sign-on (SAML)
  
  Copy from IAM Identity Center:
    Entity ID, Reply URL (ACS) → paste into Azure AD app
  
  Copy from Azure AD app:
    Federation metadata XML URL → paste into Identity Center external IdP

STEP 2 — ENABLE SCIM (Azure AD to Identity Center)
  Identity Center → automatic provisioning → copy SCIM token
  Azure AD → Enterprise App → Provisioning tab
    Mode: Automatic
    Tenant URL: <SCIM endpoint>
    Secret token: <from Identity Center>
    Mappings: sync user + group memberships
  
  Azure AD Group "aws-platform-team" → synced to Identity Center group → assigned to permission sets

STEP 3 — CONDITIONAL ACCESS
  Azure AD → Conditional Access → New Policy:
    Users: All users accessing AWS IAM Identity Center app
    Conditions: Require: MFA, Compliant device, Named location (corporate IP or VPN)
    Grant: Allow (with above conditions met)
  
  This enforces MFA AND device compliance before any AWS console access — not managed by AWS at all
```

---

## 10. How to Onboard Existing Accounts into Control Tower (Brownfield) {#brownfield}

**The Problem:**
You've just deployed Control Tower but you have 50 existing accounts outside of it.
They have no guardrails, no centralized logging, no SSO. You need to bring them in
without breaking what's already running.

---

### Brownfield Onboarding Decision Tree

```
FOR EACH EXISTING ACCOUNT, ASK:

Does it have conflicting resources that Control Tower would create?
  (CloudTrail trail named "aws-controltower-*", Config recorder, specific IAM roles)
  
  YES → PRE-CLEANUP REQUIRED
    1. Delete the conflicting Control Tower-named CloudTrail trail (CT will recreate it)
    2. Disable conflicting Config recorders
    3. Document and remove IAM roles matching "AWSControlTower*" patterns
  
  NO → SAFE TO ENROLL

Does it have an existing Organization Unit assignment?
  YES in a CT-managed OU → Can enroll directly
  YES in a non-CT OU     → Must first move to a CT-managed OU
  NO (root level)        → Move to appropriate OU, then enroll

Does it have custom SCPs that might conflict?
  → Review every SCP attached to the account or its current OU
  → Ensure they don't DENY actions Control Tower needs (CloudTrail:CreateTrail, etc.)
```

---

### Step-by-Step: Enroll an Existing Account

```bash
# STEP 1: Move account to a CT-managed OU (if not already there)
aws organizations move-account \
  --account-id 999999999999 \
  --source-parent-id r-xxxx \                  # current parent (root or non-CT OU)
  --destination-parent-id ou-xxxx-xxxxxxxx     # your CT-managed NonProd OU

# STEP 2: Enroll via Control Tower (Console recommended for first few)
  Control Tower Console → Organization → Account → Enroll
  
  OR via CLI (newer CT versions):
  aws controltower register-organizational-unit \
    --organizational-unit-id ou-xxxx-xxxxxxxx
  # This deploys CT baseline to all accounts in the OU

# STEP 3: Monitor enrollment
  Control Tower Console → Accounts → [account name]
  Status transitions: Enrolling → Enrolled (or → Failed with reason)
  
  Common failures:
    "CONFLICT_EXCEPTION" → existing CT-named resource, delete it and retry
    "ACCESS_DENIED"      → account doesn't trust the CT management account role, fix trust policy
    "TIMEOUT"            → retry; sometimes a transient CT issue

# STEP 4: Post-enrollment validation
  aws cloudtrail describe-trails \
    --trail-name-list aws-controltower-BaselineCloudTrail \
    --profile existing-account
  # Should show the trail created by Control Tower

  aws configservice describe-configuration-recorders \
    --profile existing-account
  # Should show the CT-managed Config recorder active
```

---

## 11. How to Implement Cost Governance at Scale {#cost}

**The Architecture:**

```
COST GOVERNANCE LAYERS:

Layer 1 — BUDGETS (per-account hard limits)
  Every account gets:
    Total monthly budget: $X (set in AFT account customization)
    Alert at 80%: SNS → email to account owner + Slack #aws-cost-alerts
    Alert at 100%: SNS → email + PagerDuty (if production account)
    Budget Actions at 100%: (optional) apply IAM policy to restrict new resource creation

Layer 2 — COST ANOMALY DETECTION (ML-based)
  Org-level anomaly detector:
    Monitor: all services, all accounts
    Threshold: 20% above expected spend OR > $200 anomaly
    Alert: SNS → Slack with account, service, and anomaly amount

Layer 3 — TAGGING ENFORCEMENT (SCP + Config rule)
  SCP blocks resource creation without required tags
  Config rule reports resources missing tags
  
Layer 4 — RESERVED CAPACITY GOVERNANCE
  Org-level reservation sharing (Savings Plans + RI sharing enabled)
  Reserved capacity purchased centrally in management account
  Automatically shared to all member accounts
```

---

### How to Set Up Per-Account Budgets via AFT

```hcl
# In aft-account-provisioning-framework/terraform/budgets.tf
# Runs for EVERY new account provisioned via AFT

variable "budget_limit_usd" {
  description = "Monthly budget cap from account custom_fields"
  default     = "1000"
}

resource "aws_budgets_budget" "monthly_cost" {
  name         = "monthly-cost-limit"
  budget_type  = "COST"
  limit_amount = var.budget_limit_usd
  limit_unit   = "USD"
  time_unit    = "MONTHLY"

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 80
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = ["aws-costs@company.com"]
  }

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 100
    threshold_type             = "PERCENTAGE"
    notification_type          = "FORECASTED"
    subscriber_email_addresses = ["aws-costs@company.com", "platform-oncall@company.com"]
  }
}

resource "aws_budgets_budget_action" "restrict_on_breach" {
  budget_name        = aws_budgets_budget.monthly_cost.name
  action_type        = "APPLY_IAM_POLICY"
  approval_model     = "AUTOMATIC"
  notification_type  = "ACTUAL"
  
  action_threshold {
    action_threshold_type  = "PERCENTAGE"
    action_threshold_value = 110   # At 110% of budget, apply restrictive policy
  }
  
  definition {
    iam_action_definition {
      policy_arn = aws_iam_policy.cost_restrict.arn
      roles      = ["AWSControlTowerExecution"]   # applies to the role teams assume
    }
  }
  
  subscriber {
    address           = "aws-costs@company.com"
    subscription_type = "EMAIL"
  }
}

# The restrictive policy: blocks creating expensive resources when over budget
resource "aws_iam_policy" "cost_restrict" {
  name = "BudgetBreachRestriction"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Deny"
      Action   = [
        "ec2:RunInstances",
        "rds:CreateDBInstance",
        "redshift:CreateCluster",
        "sagemaker:CreateTrainingJob"
      ]
      Resource = "*"
    }]
  })
}
```

---

## 12. How to Enforce Tagging Policy Across All Accounts {#tagging}

**The Problem:**
Without mandatory tagging, your Cost Explorer shows $500,000/month but you have no idea
which team, app, or environment is responsible for each dollar. Enforcement must happen at
both the prevention layer (SCP) and the detection layer (Config rules).

---

### Two-Layer Tagging Enforcement

```
LAYER 1 — PREVENTION (SCP: deny resource creation without required tags)
LAYER 2 — DETECTION (Config rule: flag non-compliant existing resources)
LAYER 3 — REPORTING (Cost Explorer: show spend by tag, highlight untagged)
```

```json
// SCP: Deny EC2, RDS, S3 creation without required tags
// Required tags: "owner", "environment", "cost-center", "app"
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyEC2WithoutRequiredTags",
      "Effect": "Deny",
      "Action": ["ec2:RunInstances", "ec2:CreateVolume", "ec2:AllocateAddress"],
      "Resource": [
        "arn:aws:ec2:*:*:instance/*",
        "arn:aws:ec2:*:*:volume/*"
      ],
      "Condition": {
        "Null": {
          "aws:RequestTag/owner":        "true",
          "aws:RequestTag/environment":  "true",
          "aws:RequestTag/cost-center":  "true",
          "aws:RequestTag/app":          "true"
        }
      }
    },
    {
      "Sid": "DenyRDSWithoutRequiredTags",
      "Effect": "Deny",
      "Action": ["rds:CreateDBInstance", "rds:CreateDBCluster"],
      "Resource": "*",
      "Condition": {
        "Null": {
          "aws:RequestTag/owner":       "true",
          "aws:RequestTag/environment": "true",
          "aws:RequestTag/cost-center": "true"
        }
      }
    }
  ]
}
```

```bash
# CONFIG RULE: detect existing resources missing required tags
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "required-tags-ec2",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "REQUIRED_TAGS"
    },
    "InputParameters": "{\"tag1Key\":\"owner\",\"tag2Key\":\"environment\",\"tag3Key\":\"cost-center\",\"tag4Key\":\"app\"}",
    "Scope": {
      "ComplianceResourceTypes": ["AWS::EC2::Instance","AWS::EC2::Volume","AWS::RDS::DBInstance","AWS::S3::Bucket"]
    }
  }'

# ORGANIZATIONS TAGGING POLICY (enforced via Organizations, not SCP)
# Tagging Policies validate tag values — e.g., "environment" must be one of: dev, staging, prod, sandbox
aws organizations create-policy \
  --content '{
    "tags": {
      "environment": {
        "tag_key": {"@@assign": "environment"},
        "tag_value": {"@@assign": ["dev","staging","prod","sandbox","shared"]},
        "enforced_for": {"@@assign": ["ec2:instance","s3:bucket","rds:db"]}
      }
    }
  }' \
  --description "Enforce valid environment tag values" \
  --name "TaggingPolicy-Environment" \
  --type TAG_POLICY
```

---

## 13. How to Implement Break-Glass Emergency Access {#breakglass}

**The Scenario:**
3am. Production is down. The engineer's Okta MFA device is lost. The SSO federation is broken.
Nobody can get into the AWS account. You need an emergency access path that bypasses SSO —
but is heavily audited and alarmed, so it can never be silently misused.

---

### Break-Glass Design

```
BREAK-GLASS ARCHITECTURE:

1. Emergency IAM User (one per critical account)
   Name:     BreakGlassUser-[account-name]
   Location: Each production account
   Policy:   AdministratorAccess
   MFA:      Hardware MFA token (stored in physical secure location — safe, vault)
   Password: Random 32-char, stored in Secrets Manager in Log Archive account
   
   Access keys: NONE (console only, MFA required)

2. Break-Glass Role (for programmatic emergency access)
   Name: BreakGlassRole
   Trust: Only allows assumption from break-glass IAM user (with MFA condition)
   Policy: AdministratorAccess
   Session: Max 1 hour
   
   Trust policy condition:
   "Condition": {
     "Bool": {"aws:MultiFactorAuthPresent": "true"},
     "StringLike": {"sts:RoleSessionName": "BREAKGLASS-${aws:username}-*"}
   }

3. CloudTrail Alarm on Break-Glass Usage
   Metric filter on CloudTrail logs:
     "$.userIdentity.userName = BreakGlassUser*"
   CloudWatch Alarm: fires on any match
   SNS → PagerDuty (Critical) + email to CISO + security team

4. AWS Config Rule: Break-Glass Access Keys Never Created
   Custom Config rule: if BreakGlassUser has access keys active → NON_COMPLIANT
   Auto-remediation: delete any active access keys on that user
```

---

### Step-by-Step: Create Break-Glass Access

```bash
# Run once per production account (or deploy via AFT customization)

# STEP 1: Create the break-glass IAM user
aws iam create-user \
  --user-name BreakGlassUser \
  --tags Key=Purpose,Value=EmergencyAccess Key=ManagedBy,Value=SecurityTeam

# STEP 2: Set a random password (store in Secrets Manager, NOT here)
EMERGENCY_PASSWORD=$(openssl rand -base64 32)
aws iam create-login-profile \
  --user-name BreakGlassUser \
  --password "$EMERGENCY_PASSWORD" \
  --password-reset-required

# Store in Log Archive account Secrets Manager (cross-account access only for security admins)
aws secretsmanager create-secret \
  --name "emergency/BreakGlass-app1-prod/password" \
  --description "Break glass password for app1-prod account — require 2 approvers to access" \
  --secret-string "$EMERGENCY_PASSWORD" \
  --profile log-archive-account

# STEP 3: Attach MFA requirement and AdministratorAccess
aws iam attach-user-policy \
  --user-name BreakGlassUser \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

# STEP 4: Create CloudWatch alarm on break-glass usage
# (Assumes CloudTrail logs are flowing to CloudWatch Logs group: aws-cloudtrail-logs)

aws logs put-metric-filter \
  --log-group-name aws-cloudtrail-logs \
  --filter-name BreakGlassUsageFilter \
  --filter-pattern '{ ($.userIdentity.type = "IAMUser") && ($.userIdentity.userName = "BreakGlassUser") }' \
  --metric-transformations \
    metricName=BreakGlassUsageCount,metricNamespace=SecurityAlerts,metricValue=1

aws cloudwatch put-metric-alarm \
  --alarm-name BreakGlassAccessUsed \
  --alarm-description "CRITICAL: Break-glass access used — notify security team immediately" \
  --metric-name BreakGlassUsageCount \
  --namespace SecurityAlerts \
  --statistic Sum \
  --period 60 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:us-east-1:555555555555:security-critical-alerts \
  --treat-missing-data notBreaching
```

---

## 14. How to Run a Landing Zone Health Check {#healthcheck}

**Run this quarterly, or after any major org change (new OU, bulk account enrollment, SCP change).**

---

### Health Check Runbook

```bash
#!/bin/bash
# lz-health-check.sh — Landing Zone Health Verification
# Run from management account with org admin privileges

echo "===== LANDING ZONE HEALTH CHECK ====="
echo "Date: $(date)"
echo ""

# CHECK 1: All accounts in Control Tower are Compliant
echo "[ ] CHECK 1: Control Tower Account Compliance"
aws controltower list-landing-zones \
  --query 'landingZones[0].{Status:status,DriftStatus:driftStatus}' \
  --output table

# CHECK 2: GuardDuty enabled in ALL org accounts
echo "[ ] CHECK 2: GuardDuty Coverage"
ENABLED=$(aws guardduty list-members \
  --detector-id $DETECTOR_ID \
  --query 'Members[?RelationshipStatus==`Enabled`] | length(@)' \
  --output text)
TOTAL=$(aws organizations list-accounts \
  --query 'Accounts[?Status==`ACTIVE`] | length(@)' \
  --output text)
echo "    GuardDuty enabled: $ENABLED / $TOTAL accounts"
if [ "$ENABLED" -lt "$TOTAL" ]; then
  echo "    ⚠️  WARNING: Some accounts missing GuardDuty"
fi

# CHECK 3: CloudTrail org trail is active and logging
echo "[ ] CHECK 3: Org CloudTrail Status"
aws cloudtrail get-trail-status \
  --name org-management-events \
  --query '{IsLogging:IsLogging,LastDelivered:LatestDeliveryTime}' \
  --output table

# CHECK 4: Security Hub findings — open CRITICAL count
echo "[ ] CHECK 4: Open Security Hub CRITICAL Findings"
aws securityhub get-findings \
  --filters '{
    "SeverityLabel": [{"Value": "CRITICAL", "Comparison": "EQUALS"}],
    "WorkflowStatus": [{"Value": "NEW", "Comparison": "EQUALS"}],
    "RecordState": [{"Value": "ACTIVE", "Comparison": "EQUALS"}]
  }' \
  --query 'Findings | length(@)' \
  --output text | xargs -I{} echo "    Open CRITICAL findings: {}"

# CHECK 5: SCPs attached to production OU
echo "[ ] CHECK 5: SCP Coverage on Prod OU"
PROD_OU_ID="ou-xxxx-xxxxxxxx"
aws organizations list-policies-for-target \
  --target-id $PROD_OU_ID \
  --filter SERVICE_CONTROL_POLICY \
  --query 'Policies[*].Name' \
  --output table

# CHECK 6: Log Archive bucket Object Lock still enabled
echo "[ ] CHECK 6: Log Archive S3 Object Lock"
aws s3api get-object-lock-configuration \
  --bucket org-cloudtrail-logs \
  --profile log-archive-account \
  --query 'ObjectLockConfiguration.{Status:ObjectLockEnabled,Mode:Rule.DefaultRetention.Mode,Years:Rule.DefaultRetention.Years}' \
  --output table

# CHECK 7: IAM Identity Center — no direct IAM users in prod accounts
echo "[ ] CHECK 7: IAM Users in Production Accounts (should be 0 or only BreakGlass)"
for ACCOUNT_ID in $(aws organizations list-accounts-for-parent \
  --parent-id $PROD_OU_ID \
  --query 'Accounts[*].Id' --output text); do
  USER_COUNT=$(aws iam list-users \
    --query 'Users | length(@)' \
    --output text \
    --profile "arn:aws:iam::${ACCOUNT_ID}:role/AWSControlTowerExecution" 2>/dev/null || echo "ERROR")
  if [ "$USER_COUNT" -gt "1" ]; then
    echo "    ⚠️  Account $ACCOUNT_ID has $USER_COUNT IAM users (expected: 1 break-glass only)"
  fi
done

echo ""
echo "===== HEALTH CHECK COMPLETE ====="
echo "Review any ⚠️  warnings above and remediate before next quarterly check"
```

---

## 15. Troubleshooting Common Landing Zone Problems {#troubleshooting}

```
PROBLEM: Control Tower shows account as "Drifted"
  What happened:
    Someone manually changed a Control Tower-managed resource (SCP, IAM role, CloudTrail)
    outside of Control Tower. CT detected the drift.
  
  Fix:
    Control Tower Console → Account → Re-register (runs baseline again, fixes drift)
    Do NOT manually fix the drifted resource — CT will overwrite your fix anyway.
    Preventive: SCP denying modification of CT-managed CloudTrail names

───────────────────────────────────────────────────────────────────────

PROBLEM: Account Factory account creation fails with "EMAIL_ALREADY_EXISTS"
  What happened:
    An AWS account with that email address already exists (maybe deleted but not fully purged)
  
  Fix:
    Use a different email address (e.g., aws+app1-prod-v2@company.com)
    OR: close the old account fully (takes 90 days), then reuse the email

───────────────────────────────────────────────────────────────────────

PROBLEM: SCP attached, but blocked action still works
  Diagnosis:
    1. Is the SCP actually attached to the right target (account or OU)?
       aws organizations list-policies-for-target --target-id <account-id>
    2. Is there an explicit ALLOW SCP overriding it? (unlikely but check)
    3. Is the principal exempt? (Management account root is NOT subject to SCPs)
    4. Is it a global service with a different ARN than expected?
  
  Test properly:
    aws sts assume-role --role-arn <member-account-role>   (NOT management account)
    From that session, attempt the blocked action
    Management account is always exempt from SCPs — test from a member account

───────────────────────────────────────────────────────────────────────

PROBLEM: IAM Identity Center users can't access accounts after Okta group change
  What happened:
    SCIM sync has a delay (up to 40 minutes) OR the sync failed silently
  
  Diagnosis:
    Okta Admin → Provisioning → View Logs → look for sync errors
    Identity Center → Users → check if group membership updated
  
  Fix:
    Okta: manually trigger "Sync Now" on the SCIM provisioning tab
    Identity Center: verify group exists and is assigned to the right permission set
    Workaround: manually add user to Identity Center group temporarily

───────────────────────────────────────────────────────────────────────

PROBLEM: AFT account provisioning pipeline fails at "Customizations" step
  What happened:
    Your aft-account-customizations Terraform has an error (resource limit, API error, etc.)
  
  Diagnosis:
    CodePipeline → AFT pipeline → failed stage → View in CodeBuild
    Read the Terraform error output
  
  Common causes:
    → Terraform state locked (DynamoDB lock not released) → manually delete lock item
    → Account-specific resource limit (VPC count, EIP limit) → request limit increase
    → Terraform code bug → fix in the customizations repo → pipeline retries automatically
    → IAM permission missing from AWSControlTowerExecution role → add permission, retry

───────────────────────────────────────────────────────────────────────

PROBLEM: CloudTrail logs missing for some accounts
  Diagnosis:
    aws cloudtrail get-trail-status --name org-management-events
    Check: IsLogging = true?
    Check: LatestDeliveryError (might show S3 bucket policy issue)
  
  Fix:
    S3 bucket policy in Log Archive: verify the org write permission is correct
    Specifically: "aws:PrincipalOrgID": "o-xxxxxxxxxx" must match your org ID

───────────────────────────────────────────────────────────────────────

PROBLEM: Cost Anomaly Detection fires on a legitimate workload
  What happened:
    A valid increase (quarterly report run, big training job) triggered the ML model
  
  Fix:
    Cost Anomaly Detection → Anomaly → "This was expected" (trains the model)
    OR: Create monitor with threshold filter ($1000 minimum) to reduce noise
    Long-term: tag the anomalous account/service so Cost Anomaly Detection 
              learns to separate "ML training spike" from "crypto mining"

───────────────────────────────────────────────────────────────────────

PROBLEM: "Access Denied" when trying to enroll an existing account into Control Tower
  What happened:
    The account doesn't trust the Control Tower execution role from the management account
  
  Fix:
    Login to the EXISTING ACCOUNT directly
    Go to IAM → Roles → check if AWSControlTowerExecution role exists
    If not: create it with trust for management account
    aws iam create-role \
      --role-name AWSControlTowerExecution \
      --assume-role-policy-document '{
        "Version":"2012-10-17",
        "Statement":[{
          "Effect":"Allow",
          "Principal":{"AWS":"arn:aws:iam::MANAGEMENT_ACCOUNT_ID:root"},
          "Action":"sts:AssumeRole"
        }]
      }'
    aws iam attach-role-policy \
      --role-name AWSControlTowerExecution \
      --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
    Then retry enrollment from Control Tower
```

---

## Quick Reference — Commands You'll Run Weekly

```bash
# List all accounts and their OU
aws organizations list-accounts \
  --query 'Accounts[*].{Name:Name,Id:Id,Status:Status}' \
  --output table

# Find accounts NOT in any CT-managed OU (orphans)
aws organizations list-accounts-for-parent \
  --parent-id r-xxxx \
  --query 'Accounts[*].Id' \
  --output text

# Check SCP policies attached to an OU
aws organizations list-policies-for-target \
  --target-id ou-xxxx-xxxxxxxx \
  --filter SERVICE_CONTROL_POLICY \
  --query 'Policies[*].{Name:Name,Id:Id}' \
  --output table

# Check which accounts GuardDuty is NOT enabled in
aws guardduty list-members \
  --detector-id $DETECTOR_ID \
  --query 'Members[?RelationshipStatus!=`Enabled`].{Account:AccountId,Status:RelationshipStatus}' \
  --output table

# Find all active CRITICAL Security Hub findings
aws securityhub get-findings \
  --filters '{"SeverityLabel":[{"Value":"CRITICAL","Comparison":"EQUALS"}],"WorkflowStatus":[{"Value":"NEW","Comparison":"EQUALS"}]}' \
  --query 'Findings[*].{Title:Title,Account:AwsAccountId,Resource:Resources[0].Type}' \
  --output table

# Check tag compliance across all EC2 instances in an account
aws ec2 describe-instances \
  --query 'Reservations[*].Instances[*].{ID:InstanceId,Tags:Tags}' \
  --output json | python3 -c "
import json,sys
data=json.load(sys.stdin)
for r in data:
  for i in r:
    tags = {t['Key']:t['Value'] for t in (i.get('Tags') or [])}
    missing = [t for t in ['owner','environment','cost-center','app'] if t not in tags]
    if missing:
      print(f\"{i['ID']}: missing tags: {missing}\")
"
```

---

*Last Updated: 2026-05-21 | Companion: [06_LandingZones_MultiAccount.md](./06_LandingZones_MultiAccount.md)*
*Part of: [AWS Interview Q&A Series](./00_README.md)*
