# AWS Interview Q&A — Intermediate Level

> **The Plot Thickens:** You've got the foundation. Now the questions get real.
> Intermediate interviews test whether you can *connect* services together —
> not just name them. They want to see you reason about trade-offs, debug
> failures, and design for scale. This is where most candidates stumble.

---

## Table of Contents
1. [Containers — ECS & EKS](#containers)
2. [Aurora & Database Patterns](#aurora)
3. [DynamoDB Design Patterns](#dynamodb)
4. [Messaging — SQS, SNS, EventBridge](#messaging)
5. [CloudFront & API Gateway](#edge)
6. [Auto Scaling](#scaling)
7. [Security Deep Dive](#security)

---

## 1. Containers — ECS & EKS {#containers}

---

### Q1: When would you choose ECS over EKS, and vice versa?

**The Story:**
Both run containers, but they're solving for different audiences.

**ECS (Elastic Container Service)** is AWS's native, opinionated container platform.
Think of it as a well-designed machine — it does the job, requires less expertise, and is
deeply integrated with everything AWS. You define a Task Definition (what runs), a Service
(how many), and ECS handles the rest.

**EKS (Elastic Kubernetes Service)** is AWS running Kubernetes. Kubernetes is the industry
standard, portable across clouds, deeply flexible — but it requires expertise to operate.
The learning curve is real: service meshes, ingress controllers, admission webhooks, custom operators...

```
PICK ECS WHEN:
  ✅ Team is AWS-focused, not Kubernetes-expert
  ✅ Simpler workloads: web apps, APIs, microservices
  ✅ You want deep AWS integration out of the box (IAM, ALB, CloudWatch)
  ✅ Fargate is preferred (zero node management)

PICK EKS WHEN:
  ✅ Team already knows Kubernetes
  ✅ You need workload portability (run same config on GKE / AKS too)
  ✅ Complex scheduling: GPU workloads, heterogeneous node types
  ✅ Ecosystem: Helm charts, Operators, service mesh (Istio/Linkerd)
  ✅ Large org with platform team managing K8s
```

---

### Q2: What is Fargate and what problem does it solve?

**The Story:**
Before Fargate, running containers meant you still had to manage EC2 instances — patching them,
sizing them, scaling them. You came to avoid server management, but here you are, managing servers.

Fargate removes the node entirely. You say: "Run this container with 0.5 vCPU and 1 GB RAM."
AWS finds space, provisions it, runs your container, and you never think about the host.

**Trade-offs:**
- No persistent storage on the container host (use EFS for shared storage or S3)
- Slightly more expensive than equivalent EC2 (but no ops overhead to account for)
- Cold start slightly slower (no pre-warmed nodes)
- Good for: bursty workloads, batch jobs, APIs, anything stateless

---

### Q3: How does EKS networking work with the AWS CNI plugin?

**The Story:**
Standard Kubernetes networking assigns a pod an IP from a separate CIDR range (not your VPC).
Traffic goes through NAT to reach VPC resources. AWS CNI (Container Network Interface) breaks this.

With AWS CNI, every pod gets a real VPC IP address. Your pod IP `10.0.2.45` is a real ENI
secondary IP on the worker node. This means:
- Pods can directly communicate with any VPC resource (RDS, ElastiCache, other VPCs via peering)
- No NAT overhead — direct routing
- Security Groups work on pods (SG for Pods feature)
- But — you need enough IPs in your subnets. Large clusters need large VPC CIDRs.

**Interview Follow-up Often Asked:** "What happens when you run out of IPs?"
Answer: Pods go pending. Solution: design VPC with /16 or larger; use VPC CNI prefix delegation
(each ENI gets a /28 block instead of single IPs, massively increases pod density).

---

### Q4: Explain IRSA — IAM Roles for Service Accounts.

**The Story:**
Your pod needs to read from S3. The old (bad) way: put AWS access keys in environment variables
or Kubernetes secrets. The problem: every developer can `kubectl exec` into the pod and print
the keys. Keys rotate poorly. Audit is impossible.

IRSA is the right way. It uses OIDC (OpenID Connect):

```
EKS creates an OIDC identity provider
    → Your Kubernetes service account is linked to an IAM role
    → When the pod starts, it gets a projected token (rotates every hour)
    → AWS STS validates the token with the EKS OIDC provider
    → Returns temporary credentials for that specific IAM role
    → Pod makes S3 calls with those credentials

Result:
  - Different pods get different permissions
  - No static keys anywhere
  - Full CloudTrail audit: "Pod payment-service in namespace prod assumed role PaymentS3Reader"
  - Keys auto-expire every hour
```

---

## 2. Aurora & Database Patterns {#aurora}

---

### Q5: How does Aurora's storage architecture differ from RDS?

**The Story:**
In standard RDS, your database has one storage volume — a single EBS disk in one AZ.
When you add a standby (Multi-AZ), data is synchronously replicated to *another* EBS disk in another AZ.
You have two separate copies. Writes must go to both before confirming. Network I/O doubles.

Aurora threw all of that out and invented something new.

```
AURORA STORAGE ARCHITECTURE:

Writer Instance  ──────── writes to ─────────►  SHARED STORAGE VOLUME
                                                 │
                                         ┌───────┴────────┐
                                         │  6 copies across│
                                         │  3 AZs          │
                                         │  10 GB segments │
                                         └─────────────────┘
                                               ▲
                              ┌────────────────┘
                              │
Read Replica 1   ─────────────┤  (just reads from shared storage)
Read Replica 2   ─────────────┤  (< 10ms lag, no replication network cost)
Read Replica 3   ─────────────┘
```

**What this means for you:**
- Adding read replicas doesn't cost extra storage (they share the same volume)
- Failover is faster — no data copying, just DNS flips to a replica
- Can lose 2 of 6 copies without losing write availability; can lose 3 of 6 and still read

---

### Q6: When do you use Aurora Serverless v2?

**The Story:**
Standard Aurora needs you to pick an instance type: `db.r6g.2xlarge`. What if your workload spikes
from 10 queries/sec to 10,000 on Monday morning and back down by Friday? You'd either over-provision
(waste money) or under-provision (users suffer).

Aurora Serverless v2 scales capacity in increments of 0.5 ACUs (Aurora Capacity Units),
from 0.5 ACU to 128 ACU, in under 1 second — faster than any ALB or Auto Scaling Group.

```
Real ACU example:
  Base: 2 ACU = ~4 GB RAM
  Monday 9am traffic spike → auto-scales to 32 ACU in seconds
  Sunday midnight → scales back to 0.5 ACU (near-zero cost)

Cost: Pay per ACU-second, not for provisioned capacity
Best for: Variable workloads, dev/test, multi-tenant SaaS, new apps
Not for: Extremely latency-sensitive workloads (scale-up adds ~1s latency burst)
```

---

### Q7: Explain Aurora Global Database. What problem does it solve?

**The Story:**
A bank has customers in Europe and North America. Their core database is in `us-east-1`.
European customers experience 80-120ms latency on every read because the query has to cross
the Atlantic. Compliance says EU customer data must be readable in the EU region.

Aurora Global Database solves this:

```
PRIMARY REGION (us-east-1)
  Writer ──────────────────────────────────────────────►
          writes replicated to secondary at storage layer
          latency: < 1 second (typically ~100ms)
                                                        ▼
SECONDARY REGION (eu-west-1)
  Read-only cluster ← reads local ← EU customers (< 10ms local reads)

DISASTER SCENARIO:
  us-east-1 fails → Promote eu-west-1 to primary → RTO < 1 minute, RPO < 1 second
```

**Key interview numbers:**
- Replication lag: typically < 1 second
- RPO: < 1 second (near-zero data loss)
- RTO: < 1 minute (manual promotion) or seconds (managed failover in some setups)

---

## 3. DynamoDB Design Patterns {#dynamodb}

---

### Q8: Explain single-table design in DynamoDB. Why is it controversial?

**The Story:**
In relational databases, you have one table per entity: `users`, `orders`, `products`.
DynamoDB's best practice is often to put *all entities in one table* — users, orders, and
products together — differentiated by their key structure.

**Why?** Because DynamoDB charges for each read/write. If getting a user + their last 5 orders
requires 2 separate table reads, you pay twice and make 2 round trips. With single-table design,
one query fetches both because they're co-located by partition key.

```
SINGLE TABLE EXAMPLE:
  PK              | SK               | DATA
  USER#123        | PROFILE          | {name: "Alice", email: ...}
  USER#123        | ORDER#2024-01-15 | {total: $50, status: ...}
  USER#123        | ORDER#2024-02-10 | {total: $120, status: ...}
  PRODUCT#456     | METADATA         | {name: "Widget", price: ...}

Query: "Give me user 123 and all their orders"
  → KeyConditionExpression: PK = "USER#123"
  → Returns profile + all orders in ONE request
```

**Why it's controversial:**
- Requires upfront access pattern planning — if you missed a query pattern, you may need to
  restructure the entire table
- Hard to understand for teams from relational backgrounds
- For simple apps, the complexity isn't worth it

---

### Q9: What is the DynamoDB hot partition problem and how do you fix it?

**The Story:**
DynamoDB distributes data across partitions based on your partition key. Each partition handles
up to 3,000 RCU/second and 1,000 WCU/second. If all writes go to the same partition key value,
you hit a ceiling regardless of how much capacity you've provisioned.

**Classic Example — Wrong Way:**
An e-commerce site tracks order status with `PK = "STATUS"` values: `PENDING`, `PROCESSING`, `SHIPPED`.
During a sale, 10,000 orders per second are written with `PK = "PENDING"`. All go to one partition. It's a wall.

**Fix — Write Sharding:**
```
INSTEAD OF:
  PK = "PENDING"        → ALL writes hit one partition

USE:
  PK = "PENDING#" + random(1,10)  → writes spread across 10 partitions
  → "PENDING#1", "PENDING#4", "PENDING#7"...

When reading:
  → Query all 10 shards in parallel, aggregate results in application layer
```

**Other Fix — Time-based partition keys:**
```
PK = "ORDERS#2024-01-15"  → today's orders
PK = "ORDERS#2024-01-14"  → yesterday's
→ Naturally distributes across time windows
```

---

### Q10: DynamoDB Streams — what are they and what are they good for?

**The Story:**
Every write to DynamoDB (insert, update, delete) can produce an event in DynamoDB Streams —
a 24-hour log of every change, in order, per partition key.

Think of it as the CDC (Change Data Capture) mechanism built into DynamoDB.

**Common Patterns:**
```
1. Trigger Lambda on every change:
   DynamoDB Streams → Lambda → send notification / update search index / audit log

2. Replicate to another service:
   DynamoDB Streams → Lambda → OpenSearch (keep search index in sync)
   DynamoDB Streams → Lambda → Redshift (analytics pipeline)

3. Materialized views:
   Table A changes → Streams → Lambda → update aggregate in Table B

4. Event sourcing:
   Every state change is an event → replay stream to rebuild state at any point
```

---

## 4. Messaging — SQS, SNS, EventBridge {#messaging}

---

### Q11: When do you use SQS vs SNS vs EventBridge? Give a real scenario.

**The Story:**
These three services are often confused because they all "send messages." But they solve
fundamentally different problems.

**SQS = Queue. One sender, one receiver (or a pool of competing receivers).**
*Scenario:* User uploads a video. You want to process it (transcode to multiple resolutions).
Put a message in SQS: `{videoId: "123", bucket: "uploads"}`. A pool of workers pulls from the queue.
If processing fails, the message stays and gets retried. The queue is a buffer and retry mechanism.

**SNS = Fan-out. One sender, many receivers simultaneously.**
*Scenario:* Order placed. You need to: send confirmation email, update inventory, notify the warehouse,
log the analytics event — all at the same time. SNS sends to all 4 subscribers in parallel.
Fire-and-forget; no retry mechanism if a subscriber fails.

**EventBridge = Routing by rules. One sender, filtered receivers.**
*Scenario:* Your app emits events like `{source: "payments", type: "payment.failed", amount: 5000}`.
Rule 1: If type = payment.failed AND amount > 1000 → route to fraud-detection Lambda.
Rule 2: If source = payments → route to analytics.
EventBridge is a smart router, not a queue or broadcaster.

```
DECISION TREE:
  Need to decouple and retry?     → SQS (+ DLQ for failures)
  Need to fan-out to N services?  → SNS
  Need routing based on content?  → EventBridge
  Need all three?                 → SNS → SQS (fan-out then queue per subscriber)
```

---

### Q12: Explain SQS visibility timeout and the DLQ (Dead Letter Queue).

**The Story:**
When a worker picks up an SQS message to process it, the message doesn't disappear immediately.
It becomes "invisible" to other workers for a duration called the **visibility timeout**. If the
worker finishes successfully, it deletes the message. If it crashes, the message reappears after
the timeout for another worker to try.

**Problem:** What if the message is always crashing workers? (Maybe it's malformed data.)
It loops forever, taking up a worker every few minutes.

**Solution: Dead Letter Queue (DLQ)**
```
Main Queue
  Message fails → returns to queue → tried again
  Message fails → returns to queue → tried again
  Message fails → returns to queue → tried again (maxReceiveCount = 3)
  Message fails → MOVED TO DLQ ← human investigation, CloudWatch alarm

Best Practices:
  - Set maxReceiveCount to 3-5 (not 1 — transient failures need retry)
  - Alarm on DLQ depth > 0 (something is always wrong if messages arrive here)
  - DLQ retention longer than main queue (time to investigate)
  - Redrive: fix the bug, then replay from DLQ → main queue
```

---

### Q13: What is SQS FIFO vs Standard, and what are the limits?

| | Standard Queue | FIFO Queue |
|---|---|---|
| Throughput | Unlimited (thousands/sec) | 300 TPS (3,000 with batching) |
| Ordering | Best-effort (not guaranteed) | Strictly ordered per MessageGroupId |
| Delivery | At-least-once (duplicates possible) | Exactly-once (deduplication built-in) |
| Use when | Order doesn't matter, max throughput | Payment processing, database updates, anything where order matters |
| MessageGroupId | N/A | Groups messages for ordered processing |

**Interview Trap:** "Can FIFO queues guarantee order across ALL messages?"
Answer: Only within a MessageGroupId. Different groups process in parallel, independently ordered.
Think of it like multiple checkout lanes — each lane is ordered, but you're in exactly one lane.

---

## 5. CloudFront & API Gateway {#edge}

---

### Q14: How does CloudFront caching work and how do you control it?

**The Story:**
CloudFront has 400+ edge locations worldwide. When a user in Tokyo requests your London-based app,
without CloudFront the request travels 10,000km. With CloudFront, if the response is cached at
the Tokyo edge, the user gets it in < 5ms.

**Cache behavior controlled by:**

```
1. Cache-Control headers from your origin:
   Cache-Control: max-age=86400          → cache for 24 hours
   Cache-Control: no-cache, no-store     → never cache (dynamic content)

2. CloudFront Cache Policy (newer, recommended):
   → Define which headers/cookies/query strings matter for cache keys
   → Fewer cache key components = higher hit rate

3. CloudFront TTL settings:
   Minimum TTL / Maximum TTL / Default TTL
   → Override origin headers if needed

4. Invalidation:
   aws cloudfront create-invalidation --paths "/images/*"
   → Costs $0.005/path after first 1000/month
   → Better solution: versioned URLs (app.v2.min.js)
```

**When NOT to cache:** API responses with user-specific data. Use `Vary: Authorization` header
or better — don't cache that path at all (set max-age=0).

---

### Q15: What's the difference between ALB and API Gateway?

**The Story:**
Both sit in front of your application and route HTTP traffic. But they're designed for different jobs.

| | Application Load Balancer | API Gateway |
|---|---|---|
| Primary job | Load balance to EC2/ECS/EKS backends | Managed API layer for Lambda + HTTP backends |
| Pricing | Per hour + LCU (capacity units) | Per request + data transfer |
| Throughput | Very high (millions of requests/sec) | High but throttled by default (10K req/sec, increase-able) |
| Auth | Limited (Cognito auth via rules) | Built-in: API keys, Cognito, Lambda authorizers, JWT |
| Features | Path routing, host routing, WAF | Stage management, usage plans, request/response transforms |
| WebSocket | ✅ | ✅ (API GW native WebSocket support) |
| Use for | Microservices on ECS/EKS, any HTTP backend | Serverless APIs, Lambda backends, API products |

---

## 6. Auto Scaling {#scaling}

---

### Q16: Explain the three types of Auto Scaling policies.

**The Story:**
Auto Scaling watches metrics and adds/removes capacity. But *when* it acts and *how* depends on your policy type.

**1. Target Tracking (simplest, recommended for most)**
"Keep CPU at 60% — add or remove instances automatically."
The scaling group watches the metric and adjusts to hit your target. Like a thermostat.
```
TargetTrackingConfig:
  TargetValue: 60
  PredefinedMetricType: ASGAverageCPUUtilization
```

**2. Step Scaling (reactive, tiered)**
"If CPU > 70%, add 2. If CPU > 90%, add 5 immediately."
You define steps — as the alarm gets worse, the response gets bigger.
Good for: bursty traffic where you want gradual then aggressive scaling.

**3. Scheduled Scaling (predictable)**
"Every Monday 8am, set min capacity to 20. Every Friday 8pm, set it back to 5."
You know when traffic arrives — pre-scale before it hits.
Good for: known traffic patterns (business hours, marketing campaigns, weekly batch jobs).

**Warm-up Period:** Newly launched instances need time to initialize. Set the "instance warmup"
period (e.g., 300 seconds) so new instances don't get counted in metrics before they're ready.
Otherwise Auto Scaling sees a still-high CPU and adds *more* instances unnecessarily.

---

### Q17: What is the difference between horizontal and vertical scaling?

| | Horizontal Scaling (Scale Out) | Vertical Scaling (Scale Up) |
|---|---|---|
| What you do | Add more instances | Make the instance bigger |
| AWS service | Auto Scaling Groups | Change EC2 instance type |
| Downtime | None (add/remove gradually) | Brief restart (for EC2) |
| Limits | Almost unlimited (add 1000 instances) | Limited (largest EC2 instance: 192 vCPU, 1.5 TB RAM) |
| State | Works best for stateless apps | Works for stateful apps (DB) |
| Cost | Pay for many small | Pay for one big |

**Interview Talking Point:** Modern architectures prefer horizontal scaling because it:
1. Has no upper limit
2. Provides redundancy (one node fails, others continue)
3. Enables rolling deployments without downtime

Databases often use vertical scaling (upgrade instance type) for the primary, combined with
horizontal read replicas for read workloads.

---

## 7. Security Deep Dive {#security}

---

### Q18: How do you rotate database credentials securely in a containerized app?

**The Story:**
A hard-coded database password in your code is a ticking time bomb — leaked in git history,
visible to anyone who can exec into the container, impossible to rotate without a deployment.

**The Right Way with Secrets Manager:**

```
1. Store the secret:
   aws secretsmanager create-secret \
     --name "prod/myapp/db-password" \
     --secret-string "supersecretpassword"

2. Grant access via IAM Role (IRSA for EKS, Task Role for ECS):
   Policy: secretsmanager:GetSecretValue on specific secret ARN

3. Application code:
   const secret = await secretsManager.getSecretValue({
     SecretId: 'prod/myapp/db-password'
   });

4. Enable automatic rotation:
   → Secrets Manager Lambda rotator updates the secret AND the database
   → App calls GetSecretValue on next request — gets new password
   → Zero downtime rotation
```

**For Kubernetes:** Use the Secrets Store CSI Driver + AWS Provider. It mounts the secret
as a file in the pod — even more secure because the secret never touches etcd.

---

### Q19: Explain the difference between KMS CMK and AWS-managed keys.

| | AWS Managed Keys | Customer Managed Keys (CMK) |
|---|---|---|
| Created by | AWS automatically | You create them |
| Control | AWS controls rotation (annual) | You control rotation, deletion, policy |
| Cost | Free | $1/month/key + $0.03/10K API calls |
| Audit | CloudTrail shows KMS usage | CloudTrail with full detail |
| Cross-account | No | Yes — share with other accounts |
| BYOK | No | Yes (import your own key material) |
| Use when | Default encryption, less sensitive | Regulated data, need key policy control |

**The KMS Key Policy Story:**
Even if an IAM user has `kms:Decrypt` in their identity policy, they CANNOT use the key
unless the KMS key policy also grants them access. Key policy + IAM policy = BOTH must allow.
This is why cross-account KMS access works: account A's key policy grants account B's role,
and account B's IAM policy grants the specific user/role.

---

## Quick Reference — Intermediate Gotchas

```
✅ ECS Task Role = IAM role assumed by the container (for AWS API calls)
❌ Don't put AWS keys in container environment variables

✅ Aurora read replicas use the SAME storage — adding replicas doesn't double storage
❌ Not like RDS read replicas which have their own separate storage volume

✅ DynamoDB On-Demand mode: pay per request, scales to zero
❌ DynamoDB Provisioned mode: pay even when idle, but 2.5x cheaper at steady load

✅ SNS → SQS fan-out: each subscriber gets their own queue (independent retry)
❌ SNS direct to Lambda: Lambda fails = message lost (no retry like SQS)

✅ CloudFront Origin Access Control (OAC): preferred for S3 origins
❌ OAI (Origin Access Identity): legacy, still works but OAC is the current best practice

✅ API Gateway throttling: 10K req/sec default, adjustable per stage/method
❌ Not unlimited — large spikes need limit increase request or ALB for raw throughput
```

---

*Next: [03_Advanced.md](./03_Advanced.md) — Multi-region, DR, RPO/RTO, and enterprise patterns*
