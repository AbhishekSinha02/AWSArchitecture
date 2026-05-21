# AWS Interview Q&A — DevOps, CI/CD & Infrastructure as Code

> **The DevOps Story:** In 2010, deploying software meant booking a change window,
> notifying 12 teams, and manually pushing code to servers while hoping nothing broke.
> Today, companies deploy hundreds of times a day with zero downtime. That transformation
> happened through CI/CD pipelines, infrastructure as code, and the DevOps culture shift.
> This section covers the tools and practices that make it real on AWS.

---

## Table of Contents
1. [CI/CD Pipeline Design](#cicd)
2. [AWS Native DevOps Tools](#awstools)
3. [Infrastructure as Code — CDK, CloudFormation, Terraform](#iac)
4. [Deployment Strategies](#deploy)
5. [Container DevOps — ECR, ECS/EKS Deployments](#containers)
6. [GitOps](#gitops)
7. [DevSecOps & Shift-Left Security](#devsecops)

---

## 1. CI/CD Pipeline Design {#cicd}

---

### Q1: Design a CI/CD pipeline for a microservices application on AWS.

**The Story:**
Your team has 15 microservices. A developer pushes code at 10am and wants it in production
by 10:30 without ever touching a server. Here's the pipeline that makes that happen.

```
DEVELOPER WORKFLOW:
  git push feature-branch
    │
    ▼
PULL REQUEST (GitHub / CodeCommit):
  1. Automated checks run:
     ├── Unit tests (CodeBuild)
     ├── Static analysis: SonarQube / CodeGuru Reviewer
     ├── Security scan: Snyk / Checkov (IaC scanning)
     └── Code coverage gate: must be > 80%
  2. Human review required (at least 1 approver)
  3. Merge to main branch triggers pipeline

CI PIPELINE (CodePipeline or GitHub Actions):
  Stage 1: SOURCE
    CodeCommit / GitHub → detect push to main

  Stage 2: BUILD (CodeBuild)
    ├── Run unit tests + integration tests
    ├── docker build → tag image with git SHA (e.g., myapp:abc1234)
    ├── docker push → ECR (private registry)
    ├── Run container scan: Amazon Inspector / Trivy on the image
    └── Generate build artifacts (CloudFormation templates, K8s manifests)

  Stage 3: DEPLOY TO STAGING
    ├── Update ECS task definition → new image tag
    ├── Or: Update K8s deployment manifest → ArgoCD syncs
    ├── Run smoke tests (synthetic canary tests against staging endpoint)
    └── Run DAST (Dynamic Application Security Testing) — OWASP ZAP

  Stage 4: APPROVAL GATE (optional for prod)
    → Manual approval or automatic if all tests pass + hours between 9am-5pm

  Stage 5: DEPLOY TO PRODUCTION
    ├── Blue/Green or Canary deployment (CodeDeploy or ArgoCD Rollouts)
    ├── Monitor: CloudWatch alarms on 5xx error rate, latency
    └── Auto-rollback if alarm triggers within 10 minutes of deploy
```

---

### Q2: How do you handle secrets in a CI/CD pipeline?

**The Story:**
The worst thing you can do in a pipeline is write `export DB_PASSWORD=prod_secret_123` in
a shell script that gets logged, stored in build artifacts, and accidentally committed to git.
It happens more often than you think.

**The Right Approach:**

```
BUILD TIME SECRETS (e.g., npm token, SonarQube key):
  → Store in AWS Secrets Manager or SSM Parameter Store (SecureString)
  → CodeBuild environment variable references the secret ARN, not the value
  → CodeBuild IAM role has permission to GetSecretValue
  → Never logged, never in artifacts

  buildspec.yml:
    env:
      secrets-manager:
        NPM_TOKEN: "arn:aws:secretsmanager:us-east-1:123:secret:npm-token"
    phases:
      pre_build:
        commands:
          - echo "//registry.npmjs.org/:_authToken=${NPM_TOKEN}" > ~/.npmrc

RUNTIME SECRETS (DB passwords, API keys your app needs):
  → Secrets Manager + IRSA (EKS) or Task Role (ECS)
  → App fetches secret at startup or uses SDK to fetch+cache
  → Never pass as environment variables in task definitions (visible in console)

GITHUB ACTIONS:
  → GitHub Secrets (encrypted) → reference as ${{ secrets.MY_SECRET }}
  → Use OIDC federation instead of long-lived IAM access keys:
     GitHub Action assumes IAM role via OIDC (no keys stored anywhere)
```

---

## 2. AWS Native DevOps Tools {#awstools}

---

### Q3: What are the four main AWS DevOps tools and what does each do?

**The Four Musketeers:**

**CodeCommit** — Git hosting on AWS
- Fully managed, private Git repositories
- IAM-based access (no username/password — SSH or HTTPS with IAM credentials)
- Triggers: SNS, Lambda, CodePipeline on push events
- *Reality check:* Many teams use GitHub/GitLab and just connect CodePipeline to those. CodeCommit is rarely chosen for new projects in 2024.

**CodeBuild** — Managed build server
- Spins up a container, runs your buildspec.yml, destroys the container
- Supports: Python, Node.js, Java, Docker, Go, and any language with a custom image
- Scales automatically — 10 parallel builds? No problem. No Jenkins to manage.
- Integrates natively with ECR, S3, Secrets Manager
- Pay per build minute (very cheap for small teams)

**CodeDeploy** — Deployment agent
- Deploys to EC2, ECS, Lambda, or on-premises servers
- Deployment strategies: In-place, Blue/Green, Canary, Linear
- Works with Auto Scaling Groups: terminates old instances, replaces with new
- AppSpec file defines lifecycle hooks: BeforeInstall, AfterInstall, ApplicationStart, ValidateService

**CodePipeline** — Orchestrator
- Connects Source → Build → Test → Deploy into a visual workflow
- Stages and actions: parallel actions within a stage, sequential stages
- Artifacts: S3 bucket stores output from each stage
- Integrations: GitHub, Bitbucket, Jenkins, CodeBuild, CloudFormation, ECS, Lambda

```
CODEPIPELINE FLOW EXAMPLE:

[Source: GitHub]
     │
     ▼
[Build: CodeBuild]──────────────────► [S3 Artifacts Bucket]
     │                                         │
     ▼                                         ▼
[Test: CodeBuild]                    [Deploy Staging: CodeDeploy]
     │                                         │
     ▼                                         ▼
[Manual Approval]               [Deploy Prod: CodeDeploy Blue/Green]
```

---

### Q4: What is CodeGuru and what problems does it solve?

**The Story:**
Code review at scale is hard. Senior engineers become bottlenecks reviewing every PR.
Security vulnerabilities slip through because reviewers focus on functionality, not security.

CodeGuru has two components:

**CodeGuru Reviewer** (static analysis, ML-based):
- Automatically reviews pull requests in GitHub/CodeCommit/Bitbucket
- Finds: race conditions, resource leaks, insecure coding patterns, performance anti-patterns
- Trained on Amazon's internal code — knows Java and Python deeply
- Example finding: "This S3 client is created inside a loop — move it outside to avoid connection pool exhaustion"

**CodeGuru Profiler** (runtime performance):
- Instruments your running application (JVM or Python)
- Builds a flame graph of where CPU time is spent
- Identifies: inefficient code paths, expensive regex patterns, unnecessary database calls
- "Line 247 in CartService.calculateTax() accounts for 35% of your API's CPU time"

---

## 3. Infrastructure as Code {#iac}

---

### Q5: CloudFormation vs Terraform vs AWS CDK — how do you choose?

**The Story:**
This is a religious debate in DevOps. Let's be pragmatic.

```
AWS CLOUDFORMATION — "Native AWS, declarative YAML/JSON"
  Pros:
    ✅ Native to AWS — no state file to manage (AWS keeps state)
    ✅ Rollback is automatic if a stack update fails
    ✅ Deep AWS service support (every new service supported immediately)
    ✅ AWS Config can enforce CloudFormation drift detection
  Cons:
    ❌ Verbose YAML — a simple S3 bucket is 20+ lines
    ❌ Limited logic (conditions and pseudo-functions get complex fast)
    ❌ AWS-only — can't manage GitHub repos, Datadog alerts, etc.
  Use when: AWS-only shop, compliance requires native tools, regulated environments

TERRAFORM — "Multi-cloud, HCL, community ecosystem"
  Pros:
    ✅ Multi-cloud: one tool for AWS + GCP + Azure + Kubernetes + GitHub + Datadog
    ✅ HCL is more concise and readable than CloudFormation YAML
    ✅ Massive module registry: reuse community modules (VPC, EKS, etc.)
    ✅ Plan command shows exactly what will change before it does
  Cons:
    ❌ State file: must store safely (S3 + DynamoDB locking) and protect (contains secrets)
    ❌ AWS new services lag 1-4 weeks behind CloudFormation support
    ❌ Terraform Cloud costs money for teams (open source = self-managed state)
  Use when: Multi-cloud, existing Terraform ecosystem, ops team already knows it

AWS CDK — "Infrastructure in real code (TypeScript, Python, Java...)"
  Pros:
    ✅ Write infrastructure in TypeScript/Python — IDE completion, type safety, unit tests
    ✅ Constructs: reusable, shareable components (npm packages)
    ✅ Compiles to CloudFormation — gets AWS native rollback + state management
    ✅ Loops, conditionals, abstractions — real programming constructs
  Cons:
    ❌ Synthesizes to CloudFormation — inherits CloudFormation verbosity underneath
    ❌ Learning curve: understand both CDK AND CloudFormation to debug
    ❌ Version upgrades can be breaking
  Use when: Developer-heavy teams, want reusable infra libraries, TypeScript/Python expertise
```

**My pragmatic recommendation in interviews:**
"For a greenfield AWS project, I'd choose CDK for application infrastructure (it enables developer
teams to own their infra in the same language as their app) and Terraform for org-wide resources
like VPCs, IAM, and Transit Gateway that multiple teams share — those benefit from Terraform's
planning and multi-team collaboration features."

---

### Q6: How do you structure Terraform for a large organization?

**The Story:**
A startup has one Terraform file with everything. Six months later, the team has grown to 50
engineers and a single `terraform apply` in a monolithic repo takes 45 minutes, touches production,
and requires everyone to be offline. This is the architecture that scales.

```
TERRAFORM MODULE STRUCTURE FOR ENTERPRISES:

org-terraform/
├── modules/                          # Reusable building blocks
│   ├── vpc/                          # Standard VPC with subnets, NACLs, flow logs
│   ├── eks-cluster/                  # Opinionated EKS with add-ons
│   ├── rds-aurora/                   # Aurora cluster with monitoring, backups
│   └── security-baseline/            # GuardDuty, SecurityHub, CloudTrail
│
├── environments/
│   ├── shared/                       # Transit Gateway, DNS, shared services
│   │   └── main.tf
│   ├── prod/
│   │   ├── us-east-1/
│   │   │   ├── vpc/main.tf           # Uses vpc module, prod settings
│   │   │   ├── eks/main.tf           # Uses eks module, prod node size
│   │   │   └── rds/main.tf
│   │   └── eu-west-1/               # Mirrors us-east-1 for multi-region
│   └── staging/
│       └── us-east-1/

STATE MANAGEMENT:
  backend "s3" {
    bucket         = "org-terraform-state"
    key            = "prod/us-east-1/eks/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"   # Prevents concurrent apply
    encrypt        = true
  }

PIPELINE:
  PR opened → terraform plan → post plan output as PR comment
  PR merged → terraform apply (with approval for prod)
  Never run terraform apply locally against production
```

---

## 4. Deployment Strategies {#deploy}

---

### Q7: Explain Blue/Green vs Canary vs Rolling deployments.

**The Story:**
You need to deploy a new version. The question is: how do you get from v1 to v2 without
dropping customer requests or exposing everyone to a potential bug simultaneously?

```
ROLLING DEPLOYMENT:
  Replace instances one at a time (or in batches)
  
  Before:  [v1] [v1] [v1] [v1] [v1] [v1]
  During:  [v2] [v1] [v1] [v1] [v1] [v1]   ← some traffic hits v2
  During:  [v2] [v2] [v1] [v1] [v1] [v1]
  After:   [v2] [v2] [v2] [v2] [v2] [v2]
  
  ✅ Low resource cost (no duplicate environment needed)
  ❌ v1 and v2 run simultaneously — API must be backward compatible
  ❌ Rollback is slow (must roll forward or wait for replacement)

BLUE/GREEN DEPLOYMENT:
  Run two identical environments; switch traffic at once
  
  Blue (current v1): [v1] [v1] [v1] [v1]  ← 100% traffic
  Green (new v2):    [v2] [v2] [v2] [v2]  ← 0% traffic, fully tested
  
  Switch: Route53 / ALB weight → 0% Blue, 100% Green (instant)
  Rollback: flip back to Blue (seconds)
  
  ✅ Instant rollback — just flip traffic back
  ✅ No mixed versions in production
  ❌ 2x infrastructure cost during deployment
  AWS: CodeDeploy Blue/Green for ECS; Route53 weighted for app-level

CANARY DEPLOYMENT:
  Send a small % of traffic to v2 first; watch for errors; gradually increase
  
  Phase 1:  90% v1, 10% v2   ← watch error rates for 10 min
  Phase 2:  75% v1, 25% v2   ← still healthy? proceed
  Phase 3:  50% v1, 50% v2
  Phase 4:  0% v1, 100% v2
  
  ✅ Limits blast radius — only 10% of users hit bugs in canary
  ✅ Real traffic testing (staging can never fully replicate production)
  ❌ More complex to implement and monitor
  AWS: CodeDeploy Linear10PercentEvery1Minute; 
       ALB weighted target groups; 
       AppMesh traffic management; 
       Flagger + Argo Rollouts for EKS
```

---

## 5. Container DevOps {#containers}

---

### Q8: How do you manage container images securely with ECR?

```
ECR (Elastic Container Registry) BEST PRACTICES:

1. SCANNING:
   → Enable "Enhanced Scanning" (Amazon Inspector 2): continuous, not just on push
   → Block deployment if CRITICAL vulnerabilities found
   → CodePipeline: add security gate — if Inspector finds CRITICAL severity, fail pipeline

2. IMMUTABLE TAGS:
   → Enable immutableImageTags on ECR repository
   → Once you push myapp:1.2.3, that tag can never be overwritten
   → Production deployments always reference a specific SHA tag, not "latest"
   
   BAD: image: myapp:latest  (what version is this? Changed last Tuesday? Not auditable)
   GOOD: image: myapp:a3f1b89  (git SHA — exactly reproducible, traceable)

3. LIFECYCLE POLICIES:
   → Auto-delete images older than 30 days that aren't tagged with a version
   → Keep last 10 tagged releases
   → Prevents ECR storage costs from ballooning

4. CROSS-ACCOUNT ACCESS:
   → ECR resource policy allows specific accounts to pull
   → No need to push the same image to 5 different account registries

5. ENCRYPTION:
   → ECR encrypts images with KMS CMK by default
   → Use customer-managed key for audit trail of who decrypted what
```

---

## 6. GitOps {#gitops}

---

### Q9: What is GitOps and how does it differ from traditional CI/CD?

**The Story:**
Traditional CI/CD: Code change → Pipeline runs → Pipeline calls kubectl apply → Production changes.
The pipeline *pushes* changes to production. The pipeline is the authority.

GitOps: Code change → Pipeline builds artifact, updates manifest → Git becomes the authority →
ArgoCD/Flux continuously watches Git → If cluster state != Git state → ArgoCD reconciles.
Git *pulls* the cluster into the desired state.

```
TRADITIONAL CI/CD:
  Push model: Pipeline → kubectl apply → Cluster
  Problem: 
    - What if a developer manually runs kubectl edit in production?
    - The cluster state diverges from what's in Git
    - No audit trail in Git for direct cluster changes

GITOPS (ArgoCD / Flux):
  Pull model: Git is the single source of truth
  
  Developer:
    git push → update image tag in values.yaml → PR review → merge
  
  ArgoCD (running in cluster):
    → Watches Git repo (every 3 minutes or webhook-triggered)
    → Detects: values.yaml says image should be myapp:b4c2d1
    →          current cluster has myapp:a3f1b89
    → Syncs: applies the change to match Git
    → If someone does kubectl edit manually: ArgoCD reverts it within 3 min
  
  Benefits:
    ✅ Git log IS your deployment history
    ✅ Rollback = git revert → ArgoCD syncs old version automatically
    ✅ Drift detection: any manual change gets caught and reverted
    ✅ Multi-cluster from one Git repo
    ✅ Disaster recovery: new cluster + point ArgoCD at same Git repo = restored
```

---

## 7. DevSecOps & Shift-Left Security {#devsecops}

---

### Q10: What does "shift-left security" mean in a CI/CD context?

**The Story:**
Old model: Build the feature → ship to production → security team scans it → finds SQL injection →
developer has moved on to 3 other features → fixing it is now painful and slow.

Shift-left: Move security checks as early in the process as possible — into the developer's
IDE, into the PR check, into the build pipeline — before the code ever reaches production.

```
SHIFT-LEFT SECURITY LAYERS:

Layer 0 — IDE:
  → IDE plugins: SonarLint, Snyk, AWS CodeWhisperer security scan
  → Developer sees "this looks like SQL injection" while writing code

Layer 1 — Pre-commit (git hooks):
  → Detect secrets: git-secrets, detect-secrets, Trufflehog
  → Prevents committing: AWS keys, private keys, passwords
  → Runs in < 1 second (local, no network)

Layer 2 — Pull Request (CodeBuild / GitHub Actions):
  → SAST: Static Application Security Testing (Checkmarx, Semgrep, CodeGuru)
  → SCA: Software Composition Analysis — scan dependencies for CVEs (Snyk, OWASP Dependency-Check)
  → IaC scanning: Checkov, tfsec, cfn-nag — catch "S3 bucket is public" in Terraform before it deploys
  → Block merge if: Critical vulnerability found, hardcoded secret detected

Layer 3 — Build (CodeBuild):
  → Container image scanning: Amazon Inspector, Trivy
  → Block pipeline if CRITICAL CVE in base image

Layer 4 — Deploy (Pre-production):
  → DAST: Dynamic Application Security Testing (OWASP ZAP against staging endpoint)
  → Penetration-test-like automated scan of running app

Layer 5 — Production (Runtime):
  → AWS WAF: block OWASP Top 10 (SQLi, XSS, etc.)
  → GuardDuty: detect anomalous behavior (unusual API calls, crypto mining, data exfiltration)
  → Macie: detect sensitive data in S3
  → Security Hub: aggregate all findings, centralized view
```

---

## Pipeline Quick Reference

```
CODEBUILD buildspec.yml STRUCTURE:
  version: 0.2
  env:
    secrets-manager:
      MY_SECRET: "secret-arn"
  phases:
    install:    → install dependencies
    pre_build:  → login to ECR, run pre-checks
    build:      → run tests, build artifact
    post_build: → push to ECR, update manifests
  artifacts:   → what to pass to next CodePipeline stage

CODEDEPLOY appspec.yml (ECS):
  version: 0.0
  Resources:
    - TargetService:
        Type: AWS::ECS::Service
        Properties:
          TaskDefinition: <TASK_DEFINITION>
          LoadBalancerInfo:
            ContainerName: "my-container"
            ContainerPort: 8080
  Hooks:
    - BeforeAllowTraffic: "LambdaFunctionToValidateBeforeTrafficShift"
    - AfterAllowTraffic: "LambdaFunctionToValidateAfterTrafficShift"

KEY METRICS FOR PIPELINE HEALTH:
  Deployment Frequency: How often do you deploy? (Elite: multiple/day)
  Lead Time: PR merged → production? (Elite: < 1 hour)
  Change Failure Rate: % of deployments causing incident? (Elite: < 5%)
  Mean Time to Recovery: Incident detected → resolved? (Elite: < 1 hour)
  (These are the DORA metrics — know them for any senior DevOps interview)
```

---

*Next: [05_Networking_HubSpoke.md](./05_Networking_HubSpoke.md) — Transit Gateway, Direct Connect, and Hub-Spoke architecture*
