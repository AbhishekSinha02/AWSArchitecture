# AWS Interview Q&A — Advanced Level

> **The Senior Architect Test:** At this level, interviewers don't want textbook answers.
> They want judgment. They want to hear you reason through trade-offs, explain why one
> approach beats another under specific constraints, and describe what you'd do when
> everything goes wrong at 2am. This section separates the seniors from the principals.

---

## Table of Contents
1. [Multi-Region Architecture](#multiregion)
2. [Disaster Recovery & RPO/RTO](#dr)
3. [Advanced DynamoDB & Database Patterns](#db)
4. [Event-Driven & Serverless at Scale](#serverless)
5. [Observability & Troubleshooting](#observability)
6. [AI/ML Architecture on AWS](#aiml)
7. [Cost Architecture](#cost)

---

## 1. Multi-Region Architecture {#multiregion}

---

### Q1: Design a global, active-active architecture for a financial application.

**The Story:**
The CTO says: "We cannot afford more than 5 seconds of downtime per year. We have customers
in the US, EU, and APAC. A full region failure should be invisible to users."

This is the highest bar in AWS architecture: active-active multi-region.

```
GLOBAL ACTIVE-ACTIVE ARCHITECTURE:

        Users
          │
    ┌─────▼──────┐
    │ Route53    │  Latency-based routing → nearest healthy region
    │ Health     │  Health checks on ALB endpoints
    └─────┬──────┘
          │
   ┌──────┼──────┐
   ▼      ▼      ▼
us-east-1 eu-west-1 ap-southeast-1
   │      │      │
  ALB    ALB    ALB
   │      │      │
  EKS    EKS    EKS   (identical deployments, GitOps synced)
   │      │      │
   └──────┴──────┘
          │
   ┌──────▼──────┐
   │ Aurora      │  Global Database
   │ Global DB   │  Primary: us-east-1
   │             │  Secondaries: eu-west-1, ap-southeast-1
   └─────────────┘  < 1s replication lag, < 1min failover

   ┌─────────────┐
   │ DynamoDB    │  Global Tables (multi-master)
   │ Global      │  Same table writable in all 3 regions
   │ Tables      │  Conflict resolution: last-writer-wins
   └─────────────┘
```

**The Hard Questions That Come After This:**
- "What happens when us-east-1 fails?" → Route53 health check detects failure (< 10s), DNS updates, traffic routes to EU and APAC. Aurora eu-west-1 promoted to primary (< 1 min).
- "What about split-brain in DynamoDB?" → DynamoDB Global Tables uses last-writer-wins. Design your data model to avoid concurrent writes to the same item. Or use conditional writes.
- "How do you deploy changes?" → GitOps with ArgoCD or Flux; same Git commit triggers deployment in all regions. Canary in us-east-1 first, then promote.

---

### Q2: Explain the difference between active-passive and active-active multi-region.

```
ACTIVE-PASSIVE:
  us-east-1 ────── handles ALL traffic (active)
  eu-west-1 ────── warm standby, handles ZERO traffic (passive)

  Failover:
    us-east-1 fails → Route53 fails over to eu-west-1
    RPO: < 1 second (Aurora Global)
    RTO: 1-5 minutes (manual or automated promotion)
    Cost: ~1.5x (standby runs at reduced capacity)

ACTIVE-ACTIVE:
  us-east-1 ────── handles US traffic
  eu-west-1 ────── handles EU traffic (both simultaneously active)

  Failover:
    us-east-1 fails → Route53 shifts US traffic to EU
    RPO: near zero (DynamoDB Global Tables, Aurora Global)
    RTO: seconds (just DNS propagation for Route53)
    Cost: ~2x (both regions fully sized)

CHOOSE BASED ON:
  Active-Passive: Banking, healthcare — need DR but willing to pay for simpler ops
  Active-Active: Global consumer apps, trading platforms — need zero downtime & low latency globally
```

---

## 2. Disaster Recovery & RPO/RTO {#dr}

---

### Q3: Walk me through the four DR tiers. What does each cost and when is it appropriate?

**The Story:**
DR is a spectrum. The faster you recover, the more it costs. This is the chart that should
live in your head:

```
TIER 1 — BACKUP & RESTORE
  Strategy: S3 cross-region replication of backups; CloudFormation for infra
  RPO: hours to 24 hours
  RTO: hours (deploy from scratch in DR region)
  Cost: Minimal ($100-500/month for backup storage)
  Use for: Development environments, non-critical internal tools

TIER 2 — PILOT LIGHT
  Strategy: Core infrastructure exists in DR region but scaled to minimum
             DB replica running; app servers NOT running (start on demand)
  RPO: minutes to 1 hour
  RTO: 10-60 minutes (start instances, restore data, update DNS)
  Cost: 10-20% of production (DB + network costs)
  Use for: Mid-tier apps, 4-hour SLA acceptable

TIER 3 — WARM STANDBY
  Strategy: Full production stack in DR region at reduced scale (25-50%)
             Everything running, scaled down; DMS CDC or Aurora Global keeping data in sync
  RPO: minutes
  RTO: minutes (scale up DR region + DNS cutover)
  Cost: 40-60% of production
  Use for: Business-critical apps, 1-hour SLA required

TIER 4 — ACTIVE-ACTIVE (MULTI-SITE)
  Strategy: Full production in multiple regions simultaneously
             Traffic load-balanced geographically
  RPO: near zero
  RTO: seconds (Route53 health check + DNS propagation)
  Cost: 150-200% of single region
  Use for: Banking, e-commerce, global consumer apps — zero downtime requirement
```

---

### Q4: How do you test your DR plan? Most people never do this.

**The Story:**
Every company says they have DR. Very few have actually *tested* it under realistic conditions.
The ones that haven't discover their problems during an actual outage — the worst possible time.

**The Discipline: Game Days (Chaos Engineering)**

AWS Fault Injection Simulator (FIS) lets you inject real failures:

```
GAME DAY RUNBOOK EXAMPLE:

Pre-conditions:
  - Inform stakeholders: "We're testing DR, expect 5 min degradation in STAGING"
  - Snapshot everything before starting
  - Have rollback plan ready

Test 1: AZ failure simulation
  FIS action: Terminate all EC2/ECS tasks in us-east-1a
  Expected: Auto Scaling launches in us-east-1b and us-east-1c within 3 min
  Measure: Time to recovery, any errors in CloudWatch

Test 2: RDS failover
  FIS action: Trigger RDS Multi-AZ failover
  Expected: App reconnects within 2 min, error count spikes briefly then recovers
  Measure: P99 latency during failover, number of failed requests

Test 3: Region failover (quarterly, on staging copy)
  Manual: Update Route53 health check to force failover to eu-west-1
  Expected: Traffic shifts within 60 seconds, DR region handles load
  Measure: RTO, RPO (last transaction in primary vs. first in DR)

After each test: Publish findings, fix gaps, update runbook
```

---

## 3. Advanced DynamoDB & Database Patterns {#db}

---

### Q5: Explain CQRS (Command Query Responsibility Segregation) with AWS services.

**The Story:**
In a large e-commerce platform, you have two completely different data needs:

**Writes (Commands):** "Place this order." Needs strong consistency, transactional, atomic.
**Reads (Queries):** "Show me orders from last 30 days, filtered by status, sorted by amount." Needs flexibility, performance at scale.

The mistake: using the same database model for both. Your `orders` table gets hammered with complex
read queries that slow down order placement.

**CQRS Solution:**
```
WRITE PATH (Commands):
  API → Lambda → DynamoDB (orders table)
    ✅ Optimized for writes: simple PK design, transactional
    ✅ Strong consistency where needed (conditional writes)
    ✅ DynamoDB Streams captures every change

READ PATH (Query):
  DynamoDB Streams → Lambda → OpenSearch (search, filters, facets)
                           → Redshift (analytics, aggregations)
                           → ElastiCache Redis (hot data, user dashboards)

Result:
  - Orders written to DynamoDB in < 5ms
  - Complex queries hit OpenSearch (never touch the write database)
  - Analytics hit Redshift (columnar, optimized for aggregation)
  - Dashboard loads from Redis cache (sub-millisecond)
```

---

### Q6: When would you use Redshift vs Athena for analytics?

**The Story:**
Both query big data. But they're built for different situations.

```
ATHENA — "I need to query data that lives in S3 without building infrastructure"
  How it works: Serverless; Presto engine; queries Parquet/ORC/CSV in S3
  Pricing: $5 per TB scanned (partition your data and use columnar formats!)
  Best for:
    - Ad-hoc queries on log data, CloudTrail, cost data
    - Infrequent analytical queries
    - Query data you're not ready to move into a warehouse yet
  Watch out: $5/TB adds up fast if you query huge unpartitioned tables

REDSHIFT — "I have petabytes of structured data and 50 analysts querying it daily"
  How it works: Columnar OLAP warehouse; MPP (Massive Parallel Processing)
  Pricing: Per node-hour (RA3 nodes) or serverless (per RPU-second)
  Best for:
    - Regular, performance-sensitive BI queries
    - Pre-aggregated, transformed, curated data (the "Gold" layer)
    - Tableau, QuickSight, PowerBI dashboards with < 2s response SLA
  Watch out: Higher fixed cost; needs data loaded (or use Spectrum for S3 query)

THE COMBINATION:
  Raw logs in S3 → Athena for exploration
  Refined data in S3 → Glue ETL → Redshift (for dashboard queries)
  Redshift Spectrum → query S3 directly from Redshift for historical cold data
```

---

## 4. Event-Driven & Serverless at Scale {#serverless}

---

### Q7: How do you handle idempotency in a serverless architecture?

**The Story:**
Lambda + SQS is a powerful combination. But SQS Standard offers at-least-once delivery —
the same message may be delivered twice. If your Lambda charges a customer twice because it
processed the same payment message twice, you have a very serious problem.

**Idempotency = "Doing the same thing twice has the same effect as doing it once."**

**Implementation Patterns:**

```python
# Pattern 1: Idempotency Key in DynamoDB
def process_payment(event):
    payment_id = event['paymentId']
    
    # Try to claim this payment_id
    try:
        dynamodb.put_item(
            TableName='ProcessedPayments',
            Item={'paymentId': {'S': payment_id}},
            ConditionExpression='attribute_not_exists(paymentId)'
        )
    except dynamodb.exceptions.ConditionalCheckFailedException:
        # Already processed — return success without re-processing
        return {'statusCode': 200, 'body': 'Already processed'}
    
    # First time processing — do the actual work
    charge_customer(payment_id)
    return {'statusCode': 200, 'body': 'Processed'}
```

```
Pattern 2: AWS Lambda Powertools Idempotency (built-in)
  from aws_lambda_powertools.utilities.idempotency import idempotent

  @idempotent(persistence_store=DynamoDBPersistenceLayer(table_name="IdempotencyTable"))
  def handler(event, context):
      return process_payment(event)
  
  → Automatically deduplicates within configurable TTL
  → Stores function response so replay returns the same result
  → Handles concurrent duplicate invocations safely
```

---

### Q8: Design a serverless event processing system that handles 100K events/second.

**The Architecture:**

```
INGESTION:
  Mobile/Web clients ──► API Gateway (HTTP API) ──► Kinesis Data Streams
                         or direct to Kinesis       (100 shards × 1MB/s = 100 MB/s)

PROCESSING:
  Kinesis ──► Lambda (enhanced fan-out consumers)
              - Batch window: 500ms
              - Batch size: 1000 records
              - Parallelization: 10 concurrent batches per shard
              = 100 shards × 10 parallel = 1000 concurrent Lambda invocations

STORAGE HOT PATH (< 1s latency requirement):
  Lambda ──► DynamoDB (on-demand, auto-scales)
          ──► ElastiCache Redis (counters, real-time dashboards)

STORAGE COLD PATH (analytical):
  Kinesis ──► Kinesis Firehose ──► S3 (Parquet, partitioned by date/hour)
                               ──► Redshift (for BI queries)

MONITORING:
  Iterator Age metric in CloudWatch
  If iterator age grows → Kinesis is falling behind → add shards
```

**Interview Gotcha:** "What if one Lambda fails mid-batch?"
The entire batch is retried (not just the failed record). Design for idempotency.
Use bisect-on-error (SplitBatchOnError) to retry only the failing half of the batch.

---

## 5. Observability & Troubleshooting {#observability}

---

### Q9: How do you debug a performance issue in a distributed microservices application?

**The Story:**
P99 latency on your checkout API just jumped from 200ms to 3000ms. 8am Monday morning.
No deployments happened. Here's how you work through it.

```
STEP 1 — ISOLATE: Where is the time being spent?
  X-Ray Service Map:
    → Find which service's segment shows the latency increase
    → Is it the order service? The payment service? A downstream dependency?
  
  → Found: Payment service segment shows 2800ms added latency

STEP 2 — ZOOM IN: What changed inside that service?
  CloudWatch Container Insights / Performance Insights:
    → Payment service CPU: 10% (fine)
    → Payment service memory: 85% (elevated)
    → RDS Performance Insights: top SQL = "SELECT * FROM fraud_rules" running 2700ms
    
  → Found: A full table scan on the fraud_rules table, new query added Friday

STEP 3 — ROOT CAUSE: Why is this query slow now?
  RDS Performance Insights → query plan
    → Missing index on fraud_rules.transaction_type column
    → fraud_rules table grew from 1K to 500K rows over the weekend (data import)
    → With 1K rows, full scan was fast. With 500K, it takes 2.8 seconds.

STEP 4 — FIX:
  CREATE INDEX idx_fraud_rules_type ON fraud_rules(transaction_type);
  → Query drops to 5ms
  → P99 returns to 180ms

STEP 5 — PREVENT:
  Add RDS slow query log threshold: log_min_duration_statement = 1000ms
  Alert on P99 > 500ms (not just P50 — tail latency matters)
  Load test with production-size data before deploying new queries
```

---

### Q10: What are the key CloudWatch metrics you'd alert on in a production system?

```
COMPUTE (EC2 / ECS / Lambda):
  CPUUtilization > 80% for 5 min     → scale out or investigate
  MemoryUtilization > 85%            → OOM risk (custom metric, not built-in for EC2)
  Lambda Duration approaching timeout → function getting slow
  Lambda Errors / Throttles > 0      → immediate alert
  Lambda ConcurrentExecutions > 80%  → near account limit

DATABASE:
  RDS CPUUtilization > 70%           → needs read replicas or instance upgrade
  DatabaseConnections > 80% of max   → add RDS Proxy
  ReplicaLag > 1000ms                → replica falling behind
  FreeStorageSpace < 20%             → alert to expand before it hits 0%

NETWORK / API:
  ALB 5xxCount > 10/min             → backend errors
  ALB TargetResponseTime P99 > 1s   → latency degradation
  API Gateway 4xxError spike        → client issues or auth problems
  SQS: ApproximateAgeOfOldestMessage → queue falling behind
  SQS: NumberOfMessagesSentToDLQ > 0 → poison messages, needs investigation

BUSINESS METRICS (custom):
  Orders per minute drops 50%       → something is very wrong
  Payment failures > 1%             → revenue impact
  Login failures spike              → possible brute force attack
```

---

## 6. AI/ML Architecture on AWS {#aiml}

---

### Q11: Design a production RAG (Retrieval-Augmented Generation) system for an enterprise.

**The Story:**
A law firm has 500,000 legal documents. Lawyers need to ask natural language questions and
get accurate, cited answers. The system must be accurate, explainable, and not hallucinate.

```
INGESTION PIPELINE (runs when new documents are added):

  S3 (PDF uploads)
    → EventBridge (triggers on S3 PutObject)
    → ECS Task (document processor):
        ├── Textract: extract text from PDF (handles scanned docs)
        ├── Chunking: semantic chunking (respect paragraph boundaries)
        │            chunk size: 512 tokens, overlap: 50 tokens
        ├── Bedrock Embeddings (Titan Embeddings v2):
        │   convert each chunk to 1536-dimension vector
        └── OpenSearch Serverless (k-NN index):
            store vector + metadata (doc_id, page_num, section, date)

QUERY PIPELINE (on every user question):

  User: "What does our contract say about liability in product defects?"
    │
    ▼
  Lambda (query handler):
    1. Embed the question (Titan Embeddings) → 1536-dim vector
    2. Hybrid search in OpenSearch:
       - Vector search: k-NN top 20 by semantic similarity
       - BM25 keyword search: top 20 by keyword match
       - RRF (Reciprocal Rank Fusion): merge both lists → top 5
    3. Rerank: Cohere Rerank API → pick best 3 chunks
    4. Build prompt:
       SYSTEM: You are a legal assistant. Answer only using the provided context.
               If the answer is not in the context, say "I don't have that information."
               Always cite the document and page number.
       CONTEXT: [3 retrieved chunks with metadata]
       USER: [original question]
    5. Call Bedrock (Claude 3 Sonnet/Opus)
    6. Return answer + citations to user

GUARDRAILS:
  Bedrock Guardrails:
    - Block: requests asking to generate contracts (not our use case)
    - PII detection: don't return SSNs or personal data from documents
    - Grounding check: verify answer is supported by context (reduces hallucination)
```

**Evaluation (how you know it's working):**
```
RAGAS Framework metrics:
  Faithfulness:       Is the answer supported by retrieved context? (target > 0.85)
  Answer Relevancy:   Does the answer address the question? (target > 0.80)
  Context Recall:     Did retrieval find the right documents? (target > 0.75)
  Context Precision:  Are retrieved docs actually relevant? (target > 0.70)
```

---

## 7. Cost Architecture {#cost}

---

### Q12: Our AWS bill doubled last month. Walk me through how you'd investigate.

**The Systematic Investigation:**

```
STEP 1 — FIND THE SPIKE (Cost Explorer)
  AWS Console → Cost Explorer → Daily spend graph
  → Filter by "Linked Account" if multi-account
  → Identify which DAY the spike started
  → Filter by Service → which service jumped?
  → Found: EC2 costs doubled on Tuesday

STEP 2 — DRILL DOWN INTO EC2
  Filter by: EC2 Instance Type
  → 50 new r5.4xlarge instances appeared Tuesday
  Filter by: Usage Type
  → "RunInstances:r5.4xlarge" in us-east-1
  Filter by: Tags
  → Tag: Project = "DataScience", Owner = "ml-team"

STEP 3 — FIND THE CAUSE
  CloudTrail → filter Tuesday → EC2 RunInstances events
  → 50 instances launched by IAM role: ml-pipeline-runner
  → Triggered by: SageMaker Training Job ID xyz
  → The training job had a bug — it spawned 50 training instances instead of 5

STEP 4 — IMMEDIATE MITIGATION
  Terminate idle instances (confirm with ml-team first)
  Set Service Control Policy (SCP) or budget alert:
    → Budget alert: if EC2 spend > 120% of last week → SNS → Slack alert
    → AWS Budgets action: require approval for spend > $500/day

STEP 5 — PREVENT RECURRENCE
  Add instance type constraints in SageMaker pipeline config
  Tag all resources (mandatory tagging SCP)
  Cost Anomaly Detection (ML-based, catches spikes automatically)
  AWS Budgets: alert at 80% and 100% of monthly budget
```

---

## Advanced Scenario: The 3am Production Incident

> **Interviewer:** "You're on-call. At 3am, CloudWatch alerts you that 30% of API requests
> are returning 500 errors. Orders are failing. Walk me through your response."

**The Playbook:**

```
T+0:00 — ACKNOWLEDGE
  PagerDuty → acknowledge alert (stops escalation)
  Post in #incidents Slack channel: "Investigating 500 errors in checkout API"

T+0:02 — ASSESS BLAST RADIUS
  CloudWatch → Error count per service? Only checkout? Or all services?
  Found: checkout service only, payment service healthy

T+0:04 — RECENT CHANGES
  git log (or deployment history in CodePipeline/ArgoCD)
  → Deployment at 2:45am: checkout v2.3.1 deployed to production
  → 500s started at 2:47am

T+0:06 — ROLLBACK DECISION
  Error rate: 30% and rising
  Decision: Rollback immediately, investigate after service restored
  ArgoCD → sync to previous revision → checkout v2.3.0

T+0:10 — VERIFY RECOVERY
  CloudWatch → 500 error rate dropping
  T+0:12 → 500 errors at 0%, P99 back to baseline
  Post in #incidents: "Service restored via rollback to v2.3.0. RCA to follow."

T+0:30 — ROOT CAUSE ANALYSIS
  Diff between v2.3.0 and v2.3.1
  CloudWatch Logs (checkout v2.3.1):
    "NullPointerException: cannot read property 'id' of undefined"
    at CartService.applyDiscount() line 247
  → New feature: discount code feature assumed cart always has user_id
  → Guest checkout (no user_id) → null pointer → 500

T+1:00 — FIX & POST-MORTEM
  Fix: add null check in applyDiscount()
  Test: add test for guest checkout + discount code
  Process improvement: 
    → Staging environment should mirror guest checkout scenarios
    → Add canary deployment (deploy to 5% traffic first, monitor errors before full rollout)
```

---

## Quick Reference — Senior-Level Signals

```
✅ Distinguish between RPO and RTO clearly and map them to real costs
✅ Know that "active-active" has data consistency trade-offs (conflict resolution)
✅ Understand that DynamoDB Global Tables uses eventual consistency across regions
✅ Know Aurora Global Database failover is NOT automatic by default — requires manual promotion (or Aurora Managed Failover preview)
✅ Can explain CQRS and when it's overkill vs necessary
✅ Know the difference between metric alarms and anomaly detection alarms
✅ Can design for idempotency without being asked

❌ Don't claim zero RPO without multi-region active-active + conflict handling
❌ Don't say "just add a read replica" for write-heavy workloads
❌ Don't design for scale-out if the problem is a hot partition
❌ Don't forget that Lambda at high concurrency hits the account concurrency limit (10K default, increase via support)
```

---

*Next: [04_DevOps_CICD.md](./04_DevOps_CICD.md) — Pipelines, infrastructure as code, and DevOps tooling*
