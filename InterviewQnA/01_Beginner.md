# AWS Interview Q&A — Beginner Level

> **The Foundation Story:** Every skyscraper starts with a solid foundation.
> Before you can design multi-region, fault-tolerant, petabyte-scale systems,
> you need to *own* the basics. These questions appear in every interview —
> from junior engineer to principal architect. Get them wrong, and the
> interviewer doubts everything you say after.

---

## Table of Contents
1. [IAM — Identity & Access Management](#iam)
2. [EC2 — Compute](#ec2)
3. [S3 — Object Storage](#s3)
4. [VPC — Networking Basics](#vpc)
5. [RDS — Managed Databases](#rds)
6. [CloudWatch — Monitoring](#cloudwatch)
7. [Core Cloud Concepts](#core)

---

## 1. IAM — Identity & Access Management {#iam}

---

### Q1: What is IAM and why does it matter?

**The One-Line Answer:** IAM is the bouncer at every AWS door — it decides *who* can do *what* to *which* resource.

**The Story:**
Imagine your company just got 500 employees and they all need access to your AWS account.
Without IAM, everyone gets a master key. A junior developer accidentally deletes the production
database. The company loses millions. IAM exists so you never hand out master keys.

IAM has four main actors:
- **Users** — Real humans (or long-running service accounts) with credentials
- **Groups** — Buckets for users. "Developers", "Finance", "ReadOnly"
- **Roles** — Temporary hats that services or people wear. Lambda functions assume roles. EC2 instances assume roles.
- **Policies** — JSON permission documents that say "you CAN do X" or "you CANNOT do Y"

```json
// A simple policy — allows listing S3 buckets, nothing else
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "s3:ListAllMyBuckets",
    "Resource": "*"
  }]
}
```

**The Golden Rule:** IAM defaults to DENY everything. You must explicitly allow.

---

### Q2: What's the difference between an IAM Role and an IAM User?

**The Story:**
Think of a User as a **permanent employee badge** — it has a name, a password, and doesn't change.
Think of a Role as a **visitor badge** — temporary, scoped, assumed for a specific purpose, then returned.

| | IAM User | IAM Role |
|---|---|---|
| Who uses it | Humans, long-lived apps | AWS services, applications, cross-account |
| Credentials | Username + password + access keys | Temporary tokens (STS AssumeRole) |
| Duration | Permanent until deleted | 15 min to 12 hours |
| Best for | Developer logins, legacy CI/CD | EC2, Lambda, ECS tasks, cross-account access |

**Real example:** Your Lambda function needs to read from S3. You don't give it an IAM User with keys
(that's a security disaster — keys in code get leaked). Instead, you create an IAM Role with S3 read
permissions and *attach it* to the Lambda. Lambda assumes that role automatically on every invocation.

---

### Q3: What is the IAM policy evaluation order?

**The Story:**
When AWS receives a request, it runs through a checklist — in order. The first match wins.

```
Request arrives
    │
    ▼
1. Explicit DENY anywhere? ──────────────────────────► DENIED (game over)
    │ No
    ▼
2. AWS Organizations SCP allows it? ─────────────────► If NO → DENIED
    │ Yes
    ▼
3. Resource-based policy allows it? (S3 bucket policy, KMS key policy)
    │
    ▼
4. IAM Permission Boundary allows it?
    │
    ▼
5. Identity-based policy (user/role policy) allows it?
    │
    ▼
6. Session policy allows it? (from AssumeRole with conditions)
    │
    ▼
    Default: IMPLICIT DENY
```

**Key Interview Insight:** An explicit DENY always wins — even if another policy says ALLOW.
If your SCP denies EC2 termination, no IAM admin in any child account can override that.

---

### Q4: What is MFA and when should you enforce it?

**Short Answer:** MFA (Multi-Factor Authentication) adds a second proof of identity beyond the password.

**When to enforce:**
- Root account — ALWAYS, immediately, no exceptions
- All human IAM users with console access
- Sensitive API operations (use IAM condition: `aws:MultiFactorAuthPresent = true`)
- Cross-account role assumptions for production accounts

```json
// Force MFA for sensitive actions
{
  "Condition": {
    "BoolIfExists": { "aws:MultiFactorAuthPresent": "true" }
  }
}
```

---

## 2. EC2 — Compute {#ec2}

---

### Q5: Walk me through the EC2 purchase options and when to use each.

**The Story:**
AWS sells compute like an airline sells seats — there's always a deal, but it comes with conditions.

| Option | Price | Commitment | Use When |
|--------|-------|-----------|----------|
| On-Demand | Full price | None | Dev/test, unpredictable workloads, first launch |
| Reserved (1 or 3 yr) | Up to 75% off | 1 or 3 years | Steady-state production workloads |
| Savings Plans | Up to 72% off | 1 or 3 years | Flexible (covers EC2 + Lambda + Fargate) |
| Spot | Up to 90% off | None (can be interrupted) | Batch jobs, ML training, fault-tolerant apps |
| Dedicated Host | Premium | Per-host | Licensing (Oracle, SQL Server per-socket), compliance |

**Real-world mix for a production system:**
- Core app servers → Reserved or Savings Plans (predictable)
- Batch processing layer → Spot (interruptible, save 90%)
- Dev/test environments → On-Demand (spin up and down as needed)

---

### Q6: What's the difference between a Security Group and a NACL?

**The Story:**
Think of a Security Group as a **personal bodyguard** — it travels with the instance and remembers
conversations. A NACL is a **border checkpoint** — it checks every packet, remembers nothing.

| | Security Group | Network ACL |
|---|---|---|
| Level | Instance (ENI) | Subnet |
| State | Stateful (return traffic auto-allowed) | Stateless (must allow both directions) |
| Rules | Allow only | Allow AND Deny |
| Rule evaluation | All rules evaluated | In order by rule number, first match wins |
| Default | Deny all inbound, allow all outbound | Allow all in and out |

**Common gotcha:** You open port 443 inbound in your NACL but forget to allow ephemeral ports
(1024-65535) outbound. The request comes in, but the response can't leave. Stateless = you own both directions.

---

### Q7: What are EC2 placement groups and when do you use each type?

| Type | What it does | Use when |
|------|-------------|---------|
| **Cluster** | Packs instances close together in ONE AZ | HPC, low-latency inter-node communication (< 10 Gbps networking) |
| **Spread** | Places each instance on different hardware | Small critical VMs where one hardware failure must not take two instances |
| **Partition** | Groups into partitions, each on isolated hardware sets | Hadoop, Cassandra, Kafka — fault domain awareness |

---

### Q8: Explain instance store vs EBS.

| | Instance Store | EBS |
|---|---|---|
| Persistence | Ephemeral — data lost on stop/terminate | Persistent — survives stop/terminate |
| Speed | Extremely fast (physically attached NVMe) | Fast, but network-attached |
| Snapshots | Cannot snapshot | Yes, to S3 |
| Use for | Temp data, caches, buffers, scratch space | OS volumes, databases, anything you want to keep |

---

## 3. S3 — Object Storage {#s3}

---

### Q9: What are the S3 storage classes and how do you choose?

**The Story:**
S3 storage classes are like filing systems — the more rarely you open a drawer, the cheaper
it is to store in it, but the longer you wait to open it.

```
HOW OFTEN DO YOU ACCESS THIS DATA?
    │
    ├── Daily / hourly       → S3 Standard          (highest cost, instant access)
    ├── Don't know           → S3 Intelligent-Tiering (auto-moves between tiers, small monitoring fee)
    ├── Monthly              → S3 Standard-IA        (lower cost, retrieval fee)
    ├── Quarterly            → S3 Glacier Instant    (archive cost, millisecond retrieval)
    ├── Yearly               → S3 Glacier Flexible   (cheaper, 1-12 hour retrieval)
    └── Almost never         → S3 Glacier Deep Archive (cheapest, 12-48 hours, 7-yr compliance)
```

**Lifecycle Policy Example (CloudFormation-style thinking):**
- Day 0: Created in Standard
- Day 30: Move to Standard-IA (saved 40%)
- Day 90: Move to Glacier Instant (saved another 70%)
- Day 365: Move to Glacier Deep Archive (saved another 75%)

---

### Q10: How do you secure an S3 bucket?

**The 5-Layer Answer:**

1. **Block Public Access** — Turn on at account AND bucket level. This is the "big red panic button" — overrides any ACL or policy that tries to make something public.

2. **Bucket Policy** — JSON resource policy on the bucket. Restrict by IP, VPC endpoint, specific IAM principals.

3. **Access Control Lists (ACLs)** — Legacy mechanism. Avoid unless you specifically need cross-account object-level control.

4. **Encryption** — SSE-S3 (AWS manages keys), SSE-KMS (you control keys, audit via CloudTrail), SSE-C (you provide keys).

5. **VPC Endpoint (Gateway)** — Keep S3 traffic off the public internet entirely. Traffic never leaves AWS backbone.

**Interview Story:** A company's S3 bucket was public because a developer set a bucket policy to allow `*` during testing and forgot. The Block Public Access setting at the account level would have prevented this entirely. Enable it everywhere, always.

---

### Q11: What is S3 versioning and why does it matter?

**The Story:**
Without versioning, `aws s3 cp broken_file.txt s3://mybucket/important.txt` permanently destroys
your important file. With versioning, every write creates a new version — the old one is still there.

**Key behaviors:**
- Once enabled, cannot be fully disabled (only suspended)
- Delete creates a "delete marker" — the data is still there
- MFA Delete requires MFA to permanently delete versions (ransomware protection)
- Adds cost — old versions count toward storage billing

---

## 4. VPC — Networking Basics {#vpc}

---

### Q12: Explain the difference between a public subnet and a private subnet.

**The Story:**
In a typical 3-tier architecture, you have three neighborhoods in your VPC:

```
┌─────────────────────────────────────────────┐
│  VPC  10.0.0.0/16                           │
│                                             │
│  ┌───────────────┐                          │
│  │ Public Subnet │  ← Has a route to IGW    │
│  │  10.0.1.0/24  │    Load Balancers live   │
│  │               │    here. Internet-facing. │
│  └───────────────┘                          │
│                                             │
│  ┌───────────────┐                          │
│  │Private Subnet │  ← Routes to NAT GW     │
│  │  10.0.2.0/24  │    App servers live here │
│  │               │    Can reach internet    │
│  └───────────────┘    but internet can't    │
│                       reach them directly   │
│  ┌───────────────┐                          │
│  │Isolated Subnet│  ← No route to internet │
│  │  10.0.3.0/24  │    Databases live here   │
│  │               │    No internet at all    │
│  └───────────────┘                          │
└─────────────────────────────────────────────┘
```

**What makes a subnet "public"?** Two things together:
1. Route table has a route: `0.0.0.0/0 → Internet Gateway`
2. The resource has a public IP address assigned

If either is missing, the subnet might as well be private.

---

### Q13: What is a NAT Gateway and when do you need it?

**The Story:**
Your app server in a private subnet needs to call an external API (say, Stripe for payments).
It has no public IP. Without a NAT Gateway, it can't reach the internet at all.

NAT Gateway sits in a **public subnet** and acts as a proxy:
- App server sends request → NAT GW → Internet → Response comes back → NAT GW → App server
- The internet only ever sees the NAT Gateway's IP, never the private server

**Key facts for interviews:**
- NAT Gateway is AZ-specific — deploy one per AZ for high availability
- Managed by AWS — no patching, auto-scales
- NAT Instance (old way) — you manage it, can get cheap but becomes a SPOF
- Cost: $0.045/hr + $0.045/GB processed — often the biggest surprise on AWS bills

---

### Q14: What is the difference between an Internet Gateway (IGW) and a Virtual Private Gateway (VGW)?

| | Internet Gateway | Virtual Private Gateway |
|---|---|---|
| Connects to | Public internet | On-premises network (via VPN or Direct Connect) |
| Traffic | Any internet traffic | Private encrypted traffic from your data center |
| Attachment | One per VPC | One per VPC (or use Transit Gateway instead) |
| Use case | Public-facing apps, NAT GW dependency | Hybrid cloud connectivity |

---

## 5. RDS — Managed Databases {#rds}

---

### Q15: What does "Multi-AZ" mean in RDS and how is it different from a Read Replica?

**The Story:**
**Multi-AZ** is for **surviving disasters**. **Read Replicas** are for **handling traffic**.

```
MULTI-AZ (High Availability):
  Primary Instance (AZ-a) ──synchronous replication──► Standby (AZ-b)
  - Automatic failover in 1-2 minutes if primary fails
  - Standby is NOT queryable — it only exists for failover
  - Same endpoint — your app doesn't change connection strings

READ REPLICA (Scalability):
  Primary Instance ──asynchronous replication──► Replica 1 (AZ-b)
                                              └──► Replica 2 (another region)
  - Replicas ARE queryable — route read traffic here
  - Reduces load on primary by offloading SELECT queries
  - Can be in a different region (cross-region replica for DR)
  - Small replication lag (usually < 1 second)
```

**Interview Talking Point:** RDS Multi-AZ and Read Replica are NOT mutually exclusive. A production
database should have BOTH — Multi-AZ for HA and Read Replicas for read scaling.

---

### Q16: When would you choose Aurora over RDS?

**The Answer:**
Aurora is RDS but with a completely rewritten storage engine. When you need more than standard RDS gives you:

- **Performance:** 5x MySQL throughput, 3x PostgreSQL throughput — same queries, faster
- **Storage:** Automatically grows from 10 GB to 128 TB — no pre-provisioning
- **Replicas:** Up to 15 read replicas with < 10ms lag (RDS allows 5, with higher lag)
- **Failover:** < 30 seconds (RDS Multi-AZ is 1-2 minutes)
- **Serverless:** Aurora Serverless v2 scales to zero for dev/test environments

**When to stick with RDS:** If you're migrating an existing MySQL/PostgreSQL app and don't need
the performance boost, standard RDS is simpler and cheaper at small scale.

---

## 6. CloudWatch — Monitoring {#cloudwatch}

---

### Q17: What is CloudWatch and what are its main components?

**The Story:**
CloudWatch is the nervous system of your AWS infrastructure. Everything reports back to it,
and it can trigger action when something goes wrong.

```
CLOUDWATCH COMPONENTS:

Metrics ────────────────── Numbers over time
  CPU, NetworkIn, DiskOps... or your custom metrics (page load time, order count)

Alarms ──────────────────── Watchers on metrics
  "If CPU > 80% for 5 minutes → send SNS notification → trigger Auto Scaling"

Logs ────────────────────── Text output from your applications
  EC2 syslog, Lambda output, application traces, RDS slow query log

Log Insights ────────────── Query engine for logs
  "Show me the top 10 slowest API calls in the last hour" (SQL-like syntax)

Dashboards ──────────────── Visual displays
  Real-time graphs of metrics for NOC/operations teams

Events (EventBridge) ────── Scheduled and reactive automation
  "Run this Lambda every day at 2am" or "On EC2 instance termination, do X"
```

---

### Q18: What's the difference between CloudWatch and CloudTrail?

**The one-liner:** CloudWatch watches your **applications and infrastructure**. CloudTrail watches **who does what in your AWS account**.

| | CloudWatch | CloudTrail |
|---|---|---|
| What it captures | Metrics, logs, events from AWS services | API calls made to AWS (console, CLI, SDK) |
| Who uses it | Developers, DevOps, Ops teams | Security, compliance, audit teams |
| Example | "CPU on web server spiked at 3am" | "Someone deleted the S3 bucket at 3am — it was user Bob from IP 1.2.3.4" |
| Retention | Configurable (logs: forever if stored in S3) | 90 days in CloudTrail console; indefinite in S3 |

---

## 7. Core Cloud Concepts {#core}

---

### Q19: What is the Shared Responsibility Model?

**The Story:**
AWS and you split the security job. AWS owns the building; you own what you put inside.

```
AWS RESPONSIBILITY ("Security OF the cloud"):
  ├── Physical data centers (guards, fire suppression, power)
  ├── Network infrastructure (fiber, switches, routers)
  ├── Hypervisor (the virtualization layer)
  └── Managed service software (RDS engine patching, S3 durability)

YOUR RESPONSIBILITY ("Security IN the cloud"):
  ├── Operating system patches (your EC2 instances)
  ├── Application security (your code, your vulnerabilities)
  ├── IAM configuration (who has access to what)
  ├── Data encryption (you choose whether to encrypt)
  └── Network configuration (security groups, NACLs, VPC design)
```

**Interview insight:** The line shifts with managed services. With EC2, you patch the OS.
With Lambda, you don't — AWS patches the runtime environment. With RDS, AWS patches the
database engine, but you still control who can connect to it.

---

### Q20: What are the AWS Well-Architected Framework pillars?

**The Acronym to Remember: CROPSS** (or **SCORES**)

| Pillar | What It Asks |
|--------|-------------|
| **O**perational Excellence | "Can you run and monitor systems and improve processes?" |
| **S**ecurity | "Can you protect data, systems, and assets?" |
| **R**eliability | "Can you recover from failures and meet demand?" |
| **P**erformance Efficiency | "Are you using compute resources efficiently?" |
| **C**ost Optimization | "Are you avoiding unnecessary costs?" |
| **S**ustainability | "Are you minimizing environmental impact?" |

**In interviews:** When asked to design anything, close your answer by walking through 2-3 of these
pillars. "For security, I'd add WAF and Secrets Manager. For reliability, I'd use Multi-AZ and
Auto Scaling. For cost, I'd move infrequent data to S3-IA via lifecycle policies."
Interviewers love this — it shows you think holistically.

---

## Quick Reference — Beginner Gotchas

```
✅ Security Groups are STATEFUL — you only need one rule for request+response
❌ NACLs are STATELESS — you need rules for BOTH directions

✅ S3 bucket names are GLOBALLY unique across all AWS accounts
❌ Not just unique in your account or region

✅ IAM is GLOBAL — users/roles exist across all regions
❌ Most other resources are region-specific

✅ Default VPC comes pre-configured in every region
❌ Don't use default VPC for production — create dedicated VPCs

✅ EC2 public IP changes on stop/start (use Elastic IP to fix it)
❌ Elastic IP costs money when NOT attached to a running instance

✅ CloudWatch retains metrics for 15 months
❌ Standard resolution is 1 minute — use high-resolution (1 sec) for faster response
```

---

*Next: [02_Intermediate.md](./02_Intermediate.md) — ECS, EKS, Aurora patterns, DynamoDB design, and more*
