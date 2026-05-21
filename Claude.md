# AWS Architecture — Master Reference & Interview Mind Map

> **Purpose:** Living reference for AWS architecture, troubleshooting, networking, data migration, and enterprise use cases.  
> **Audience:** Senior Architect / Cloud Engineer
> **Update cadence:** Append new sections at bottom; tag with `## [Topic] — Update YYYY-MM-DD`

---

## 📐 HOW TO USE THIS FILE

```
1. INTERVIEW PREP  → Read the relevant DOMAIN section + USE CASE
2. ARCHITECTURE    → Use PATTERN LIBRARY to compose solutions quickly
3. TROUBLESHOOT    → Jump to TROUBLESHOOTING TREES
4. WHITEBOARD      → Use STEP SEQUENCES for live design sessions
5. DEEP DIVE       → Each section links to sub-topics you can expand
```

---

## 🗺️ MASTER MIND MAP — AWS Domains

```
AWS Architecture
├── 1. COMPUTE          → EC2, ECS, EKS, Lambda, Batch, App Runner
├── 2. NETWORKING       → VPC, Transit Gateway, Direct Connect, Route 53, CloudFront
├── 3. STORAGE          → S3, EBS, EFS, FSx, Storage Gateway
├── 4. DATABASES        → RDS, Aurora, DynamoDB, Redshift, ElastiCache, DocumentDB
├── 5. MESSAGING        → SQS, SNS, EventBridge, Kinesis, MSK (Kafka)
├── 6. SECURITY         → IAM, KMS, Secrets Manager, WAF, Shield, GuardDuty
├── 7. OBSERVABILITY    → CloudWatch, X-Ray, OpenSearch, Athena, CloudTrail
├── 8. AI / ML          → SageMaker, Bedrock, Comprehend, Rekognition, Textract
├── 9. DATA PLATFORM    → Glue, EMR, Lake Formation, DataZone, Redshift Spectrum
├── 10. MIGRATION       → DMS, MGN, DataSync, Snow Family, Transfer Family
└── 11. USE CASES       → AI, IoT, Migration, Retail, Banking, Insurance, Logistics
```

---

## 1. COMPUTE

### EC2 — Decision Tree
```
Need compute? →
  Stateless, bursty?          → Lambda (< 15 min) or Fargate (containerized)
  Long-running, stateful?     → EC2 (reserved/savings plan for cost)
  Batch/HPC workload?         → AWS Batch + Spot Instances
  Containerized microservice? → ECS Fargate or EKS (Kubernetes)
  Simple web API?             → App Runner (zero infra)
```

### EC2 Key Concepts (Interview Hot Topics)
| Concept | What to Know |
|---|---|
| Placement Groups | Cluster (low latency), Spread (fault isolation), Partition (Hadoop/Cassandra) |
| Instance Store vs EBS | Instance store = ephemeral, high IOPS; EBS = persistent, snapshots |
| Spot Interruption | 2-min notice; use with SQS + checkpointing for resilience |
| AMI Baking | Packer → Golden AMI → Auto Scaling Group; faster launch vs user-data |
| Nitro System | Hardware virtualization; enables bare-metal, ENA networking, NVMe |

### EKS — Architecture Layers
```
Control Plane (AWS-managed)
└── Worker Nodes (EC2 or Fargate)
    ├── kube-proxy           → Service routing
    ├── CoreDNS              → Internal DNS
    ├── AWS CNI Plugin       → Pods get VPC IPs (native networking)
    ├── Cluster Autoscaler   → Scales node groups
    └── Karpenter            → Next-gen node provisioning (faster, cost-aware)

Add-ons for Production:
├── AWS Load Balancer Controller → ALB/NLB ingress
├── EBS/EFS CSI Driver           → Persistent volumes
├── IRSA (IAM Roles for Service Accounts) → Pod-level AWS permissions
└── ADOT / CloudWatch Agent      → Metrics & traces
```

### Lambda — Architecture Patterns
```
Event Sources → Lambda → Destinations
├── API Gateway   (sync)   → REST / WebSocket APIs
├── SQS           (async)  → Queue consumer; handles DLQ
├── S3 Events     (async)  → Object processing pipeline
├── EventBridge   (async)  → Rule-based routing
├── Kinesis/MSK   (stream) → Real-time processing; batch windows
└── ALB           (sync)   → HTTP compute at scale

Cold Start Mitigation:
  - Provisioned Concurrency (pre-warm)
  - Lambda SnapStart (Java — snapshot restore)
  - Keep package < 50 MB, avoid VPC unless needed
```

---

## 2. NETWORKING

### VPC — Foundational Architecture
```
Region
└── VPC (CIDR: 10.0.0.0/16)
    ├── Public Subnet  (10.0.1.0/24)  → IGW route; ELB, NAT GW, Bastion
    ├── Private Subnet (10.0.2.0/24)  → App Tier; NAT GW for egress
    └── Isolated Subnet(10.0.3.0/24)  → DB Tier; NO internet route

Security Layers:
  NACLs   → Stateless; subnet-level; explicit ALLOW + DENY; numbered rules
  SGs     → Stateful; instance-level; ALLOW only; reference other SGs
```

### VPC Connectivity Patterns
| Pattern | Use Case | Key Limit |
|---|---|---|
| VPC Peering | 1:1 connection, same/cross-account | No transitive routing |
| Transit Gateway | Hub-spoke, 1000s of VPCs | Route tables per attachment |
| PrivateLink | Expose service privately | Per-endpoint costs |
| VPN (Site-to-Site) | On-prem connectivity, quick setup | 1.25 Gbps per tunnel |
| Direct Connect | Dedicated 1/10/100 Gbps to on-prem | Lead time 30-90 days |
| Direct Connect + VPN | Encrypted dedicated link | Best practice for banking |

### DNS & Routing — Route 53
```
Routing Policies:
  Simple        → Single record, no health check
  Weighted      → A/B, canary deployments (weight 0-255)
  Latency       → Route to lowest-latency region
  Failover      → Active/Passive; requires health check
  Geolocation   → Compliance (GDPR, data residency)
  Geoproximity  → Shift traffic by geographic bias
  Multi-value   → Up to 8 healthy records (NOT a load balancer)

Private Hosted Zone:
  → Associate with VPC for internal DNS resolution
  → Split-horizon: same domain, different answers inside/outside VPC

Resolver:
  → Inbound endpoint: on-prem DNS queries → Route 53
  → Outbound endpoint: Route 53 → on-prem DNS (forwarding rules)
```

### CloudFront — CDN Architecture
```
Origin Types:
  S3 Bucket        → Static assets; use OAC (Origin Access Control)
  ALB / API GW     → Dynamic content, HTTPS required
  Custom Origin    → Any HTTP endpoint

Key Features:
  - Edge Locations (400+) → Cache-Control headers drive TTL
  - Lambda@Edge / CF Functions → Auth, URL rewrites, A/B testing at edge
  - Field-Level Encryption → PII protection before reaching origin
  - WAF Integration → Block at edge, not origin

Cache Invalidation:
  Cost: $0.005/path (first 1000 free/month)
  Pattern: Versioned file names (app.v2.js) > invalidations
```

### Networking Troubleshooting Tree
```
CONNECTIVITY ISSUE?
├── Can ping public IP? No →
│   ├── Check SG: inbound rule for port/protocol from source
│   ├── Check NACL: both inbound AND outbound rules (ephemeral ports 1024-65535)
│   └── Check Route Table: correct route to IGW/NAT GW/TGW
│
├── VPC to VPC? →
│   ├── Peering: non-overlapping CIDRs? Routes added both sides? SG allows peer VPC CIDR?
│   └── TGW: attachment in correct TGW route table? Propagation enabled?
│
├── On-prem to AWS? →
│   ├── VPN: tunnel UP? BGP session? Correct pre-shared key?
│   ├── Direct Connect: BGP prefixes advertised? VLAN correct? LOA-CFA valid?
│   └── Check VGW vs TGW attachment on VPN
│
└── DNS not resolving? →
    ├── Private hosted zone associated with correct VPC?
    ├── enableDnsSupport = true on VPC?
    └── Resolver rules for on-prem? Endpoints in right subnets?
```

---

## 3. STORAGE

### S3 — Architecture Patterns
```
Storage Classes (cost vs access):
  S3 Standard           → Frequent access; 99.99% availability
  S3 Intelligent-Tiering → Auto-move based on access patterns
  S3 Standard-IA        → Infrequent access; 30-day min; retrieval fee
  S3 Glacier Instant    → Archive, millisecond retrieval
  S3 Glacier Flexible   → Archive, 1-12 hours retrieval
  S3 Glacier Deep       → 12-48 hours; cheapest storage

Key Features for Architects:
  - Versioning + MFA Delete → Data protection
  - Replication (CRR/SRR)   → DR, compliance, aggregation
  - Object Lock (WORM)      → Compliance, ransomware protection
  - S3 Event Notifications  → → SQS / SNS / Lambda / EventBridge
  - S3 Access Points        → Simplified access control per team/app
  - Transfer Acceleration   → CloudFront edge for uploads

Performance:
  - 3,500 PUT/COPY/POST/DELETE + 5,500 GET per prefix/sec
  - Use multiple prefixes to scale (date-based, hash prefix)
  - Multipart Upload: mandatory > 5 GB, recommended > 100 MB
```

### EBS vs EFS vs FSx
| | EBS | EFS | FSx for Windows | FSx for Lustre |
|---|---|---|---|---|
| Type | Block | File (NFS) | File (SMB) | HPC Parallel |
| Multi-attach | io1/io2 only | Yes (multi-AZ) | Yes | Yes |
| Use case | OS disk, DB | Shared web content | Active Directory, .NET | ML training, HPC |
| Performance | up to 256K IOPS | Bursting/Provisioned | SSD-backed | Sub-ms, GB/s |

---

## 4. DATABASES

### Database Selection Matrix
```
Workload → Right DB:
  Relational OLTP    → Aurora (PostgreSQL/MySQL) — 5x MySQL, 3x PostgreSQL perf
  Global low-latency → DynamoDB — single-digit ms, 10 TB+ tables, Global Tables
  Data Warehouse     → Redshift — columnar, RA3 nodes, Spectrum for S3 query
  Cache / Session    → ElastiCache (Redis: rich data types; Memcached: simple)
  Search             → OpenSearch — full-text, log analytics
  Document           → DocumentDB (MongoDB-compatible)
  Graph              → Neptune — social graphs, fraud detection
  Time Series        → Timestream — IoT metrics, DevOps monitoring
  Ledger             → QLDB — immutable audit log (financial records)
```

### Aurora — Key Differentiators (Interview Gold)
```
Aurora Architecture:
  Writer Instance
  └── Shared Storage Volume (6 copies across 3 AZs, 10 GB segments)
      └── Up to 15 Read Replicas (< 10 ms replica lag)

Features:
  - Aurora Serverless v2: auto-scale 0.5–128 ACU, scales in < 1 sec
  - Global Database: < 1 sec cross-region replication (RPO < 1s, RTO < 1 min)
  - Aurora ML: call SageMaker / Comprehend from SQL
  - Backtrack: rewind DB in-place (no restore) up to 72 hours
  - Parallel Query: push query processing to storage layer

vs RDS Multi-AZ:
  RDS = synchronous standby (failover 1-2 min)
  Aurora = storage-level HA, failover < 30 sec, no data loss
```

### DynamoDB — Patterns & Pitfalls
```
Key Concepts:
  Partition Key + Sort Key → composite key for 1:N relationships
  GSI (Global Secondary Index) → query on non-key attributes; own RCU/WCU
  LSI (Local Secondary Index) → alternate sort key; shares table RCU/WCU
  DAX → in-memory cache, microsecond reads

Design Patterns:
  Single-table design → one table, overloaded PK/SK (pk=USER#123, sk=ORDER#456)
  Adjacency list      → graph relationships in one table
  Sparse indexes      → GSI only indexes items with the attribute (cost saving)

Hotspot Problem:
  Bad: pk=STATUS (all writes to "ACTIVE" → hot partition)
  Fix: pk=STATUS#SHARD_N (write sharding with random suffix 1-10)

Capacity Modes:
  Provisioned → predictable traffic; auto-scaling; reserved capacity
  On-demand   → variable/unknown traffic; 2.5x more expensive but zero planning
```

---

## 5. MESSAGING & EVENTS

### Messaging Pattern Selection
```
Pattern          → Service       → Why
Fire-and-forget  → SNS           → Fan-out to N subscribers
Queue processing → SQS           → Decoupling, retry, DLQ
Event bus        → EventBridge   → Rule-based routing, SaaS integration
Real-time stream → Kinesis DS    → Ordered, replay, 365-day retention
High-throughput  → MSK (Kafka)   → Existing Kafka workloads, KSQL
Workflow         → Step Functions → Orchestrate Lambda/services with state

SQS Gotchas:
  - Visibility timeout > processing time (else duplicate processing)
  - DLQ: maxReceiveCount threshold before DLQ (set 3-5)
  - Long polling: 20 sec reduces empty receives (cost + latency)
  - FIFO: 300 TPS (3000 with batching); exactly-once; MessageGroupId for ordering
  - Standard: at-least-once; best-effort ordering; nearly unlimited TPS
```

### EventBridge — Event-Driven Architecture
```
Event Bus
├── Default bus    → AWS service events
├── Custom bus     → Your application events
└── Partner bus    → SaaS (Salesforce, Datadog, Auth0)

Rule → Pattern matching → Target
  Targets: Lambda, SQS, SNS, Step Functions, API GW, Kinesis, ECS, Batch

Pipes (new):
  Source → Filter → Enrichment (Lambda) → Target
  One-to-one event routing with transformation

Schema Registry:
  → Discover and version event schemas
  → Generate code bindings for event handling
```

---

## 6. SECURITY

### IAM — Mental Model
```
IDENTITY (Who?)                  PERMISSION (What?)
├── Users (human)           →    Policies (JSON)
├── Groups                  →      ├── Inline (attached to one entity)
├── Roles (assumed)         →      └── Managed (AWS or Customer)
└── Service Principals      →
                                 BOUNDARY
                                 ├── Permission Boundaries (max permissions)
                                 ├── SCPs (Org-level guardrails)
                                 └── Resource Policies (S3, KMS, etc.)

Evaluation Order:
  Explicit DENY → overrides everything
  → Org SCP → Resource Policy → Permission Boundary → Identity Policy → Session Policy
  Default: IMPLICIT DENY (nothing allowed unless explicitly permitted)
```

### Zero Trust on AWS
```
Principles → AWS Services:
  Verify explicitly      → Cognito, IAM Identity Center, MFA
  Least privilege        → IAM + Permission Boundaries + SCPs
  Assume breach          → GuardDuty, Security Hub, Macie
  Micro-segmentation     → SGs, NACLs, PrivateLink
  Encrypt everywhere     → KMS, ACM, TLS enforcement
  Continuous monitoring  → CloudTrail, Config, CloudWatch

KMS Key Types:
  AWS Managed (aws/s3)    → Free, AWS controls rotation
  Customer Managed (CMK)  → You control; $1/month/key; audit via CloudTrail
  External Key Material   → BYOK; revoke at any time
  CloudHSM                → FIPS 140-2 Level 3; dedicated HSM
```

### Security Troubleshooting
```
ACCESS DENIED?
├── Check IAM Policy Simulator first
├── Is there an explicit DENY? (SCP, Permission Boundary, Resource Policy)
├── Cross-account? → Both identity policy AND resource policy must allow
├── KMS encrypted resource? → Key policy must allow the IAM principal
└── Session policy restricting? (AssumeRole with policy conditions)

DATA EXPOSURE RISK?
├── S3 public access? → Block Public Access setting (account + bucket level)
├── Secrets in code?  → Secrets Manager + Lambda env var reference
├── Unencrypted EBS?  → Enforce encryption by default (account setting)
└── Macie finding?    → Automated sensitive data discovery in S3
```

---

## 7. OBSERVABILITY

### Three Pillars
```
METRICS → CloudWatch Metrics
  - EC2: CPU, Network, Disk (custom: memory requires agent)
  - Custom Metrics: PutMetricData API; 1-sec resolution (high-res)
  - Alarms: ALARM/OK/INSUFFICIENT_DATA → SNS → Lambda → Auto Scaling

LOGS → CloudWatch Logs
  - Log Groups → Log Streams → Log Events
  - Log Insights: SQL-like queries; use for ad-hoc analysis
  - Metric Filters: extract metrics from log patterns
  - Subscription Filters: → Kinesis → OpenSearch (real-time)

TRACES → X-Ray
  - Service Map: visual dependency graph
  - Segments + Subsegments: latency breakdown
  - Annotations: filterable key-value metadata
  - Groups + Sampling Rules: control trace volume

OpenSearch / Elasticsearch:
  - Log aggregation at scale
  - Full-text search on logs
  - Kibana dashboards
  - Combine with Kinesis Firehose for ingestion pipeline
```

---

## 8. AI / ML ON AWS

### AI Services — When to Use What
```
No ML expertise needed (Managed AI Services):
  Textract       → Document OCR, forms, tables extraction
  Comprehend     → NLP: sentiment, entities, PII, topic modeling
  Rekognition    → Image/video: labels, faces, text, moderation
  Transcribe     → Speech-to-text (real-time + batch)
  Translate      → Neural machine translation (75+ languages)
  Polly          → Text-to-speech
  Forecast       → Time-series forecasting (supply chain)
  Personalize    → Real-time recommendations
  Fraud Detector → Online fraud (transactions, account takeover)
  Kendra         → Intelligent enterprise search (RAG-ready)

ML Platform (SageMaker):
  Data Prep      → Data Wrangler, Ground Truth (labeling)
  Training       → Training Jobs, Experiments, Debugger
  MLOps          → Pipelines, Model Registry, Model Monitor
  Deployment     → Real-time Endpoints, Batch Transform, Serverless
  Feature Store  → Online (low-lat) + Offline (S3) feature repository

GenAI (Amazon Bedrock):
  Foundation Models: Claude (Anthropic), Llama, Titan, Mistral, Cohere
  RAG Pattern   → Knowledge Bases → S3 → Embeddings → Vector DB → Retrieve + Generate
  Agents        → Multi-step task execution with tool use
  Guardrails    → Content filtering, PII redaction
  Fine-tuning   → Continued pre-training + fine-tuning on Titan/Cohere
```

### RAG Architecture on AWS
```
Ingestion Pipeline:
  S3 Docs → Textract (PDF/image) → Bedrock Embeddings → OpenSearch / pgvector (Aurora)

Query Pipeline:
  User Query → Bedrock Embeddings → Vector Search (k-NN) → Top-K chunks
      ↓
  Prompt = System Prompt + Retrieved Chunks + User Query
      ↓
  Bedrock (Claude/Titan) → Response

Hybrid Search:
  BM25 (keyword) + Semantic (vector) → RRF (Reciprocal Rank Fusion) rerank
  Use Amazon OpenSearch hybrid query or Bedrock Knowledge Bases built-in hybrid
```

---

## 9. DATA PLATFORM

### Modern Data Architecture on AWS
```
Ingest Layer:
  Batch:      S3 → Glue → Redshift / Athena
  Streaming:  Kinesis → Lambda → S3 / DynamoDB / OpenSearch
  CDC:        DMS (Database Migration Service) → S3 / Kinesis → Redshift

Storage Layer:
  Raw (Bronze):     S3 (Parquet/ORC) → Glue Catalog
  Curated (Silver): Glue ETL → S3 → Glue Catalog
  Aggregate (Gold): Redshift / Athena → QuickSight

Governance:
  Lake Formation  → Fine-grained column/row access control
  DataZone        → Data marketplace, business glossary, discovery
  Macie           → PII detection in S3
  Glue DataBrew   → Visual data profiling + transformation

Query Engines:
  Athena          → Serverless SQL on S3 (Presto); pay per query
  Redshift Spectrum → Query S3 from Redshift; no data movement
  EMR             → Spark, Hive, Flink on managed clusters
```

---

## 10. MIGRATION

### Migration Strategy — 7 R's
```
RETIRE     → Decommission; not needed in cloud
RETAIN     → Keep on-prem (compliance, latency, not yet)
REHOST     → Lift-and-shift; AWS MGN (Application Migration Service)
REPLATFORM → Lift-tinker-shift; move DB to RDS, app to Elastic Beanstalk
REPURCHASE → Move to SaaS (Salesforce, Workday, ServiceNow)
REFACTOR   → Re-architect for cloud-native (microservices, serverless)
RELOCATE   → VMware Cloud on AWS; vSphere on AWS
```

### Database Migration
```
DMS (Database Migration Service):
  ├── Source: Oracle, SQL Server, MySQL, PostgreSQL, MongoDB, S3, etc.
  ├── Target: Aurora, DynamoDB, Redshift, OpenSearch, S3, etc.
  ├── Full Load → CDC (Change Data Capture) for near-zero downtime
  └── Schema Conversion Tool (SCT): heterogeneous migration (Oracle → Aurora PG)

Steps for Zero-Downtime Migration:
  1. SCT: convert schema + stored procedures
  2. DMS Full Load: copy existing data
  3. DMS CDC: capture ongoing changes (binlog / redo log)
  4. Validation: row counts, checksums, application-level testing
  5. Cutover: update connection strings, DNS cutover
  6. Rollback: DMS reverse task if issues

Common Issues:
  LOB (Large Object) columns → use LOB mode in DMS; perf impact
  Character encoding         → UTF-8 mismatch causes truncation
  Stored procedures          → SCT conversion report; manual effort
  Sequences / Identity cols  → Need manual handling in target
```

### High-Volume Data Migration
```
Volume Decision Tree:
  < 10 TB, fast internet   → AWS DataSync (NFS/SMB → S3/EFS/FSx)
  10 - 80 TB               → Snowball Edge Storage Optimized (80 TB usable)
  80 TB - 100 PB           → Multiple Snowballs or Snowball Edge Compute
  > 100 PB or exabyte      → Snowmobile (truck; 100 PB per trip)
  Ongoing transfer         → DataSync + Direct Connect (dedicated bandwidth)
  Database streaming       → DMS CDC (continuous; low latency)

DataSync Architecture:
  On-Prem NAS/NFS
    └── DataSync Agent (VM/hardware)
        └── DataSync Service (AWS-managed)
            └── S3 / EFS / FSx / S3 Glacier
  
  Features:
    - Automatic encryption in transit (TLS) + at rest (KMS)
    - Data integrity verification (checksums)
    - Bandwidth throttling (avoid saturating WAN)
    - Scheduled tasks (off-peak transfers)
    - CloudWatch metrics per task

Snowball Edge — Enterprise Workflow:
  1. Order via console → device shipped in 1-2 days
  2. Connect to on-prem switch (10 GbE)
  3. Install OpsHub (GUI) or use CLI
  4. Copy data: `snowball cp -r /source s3://bucket/prefix`
  5. Ship back → AWS ingests to S3 (10-15 days total)
  6. Verify: S3 checksums match Snowball manifest
```

---

## 11. USE CASES — ENTERPRISE PATTERNS

### 🤖 AI / GenAI Architecture
```
Pattern: Intelligent Document Processing (Banking / Insurance)
  S3 (PDF/images)
    → Textract (OCR, forms, tables)
    → Lambda (normalize JSON)
    → S3 (structured data)
    → DynamoDB (index metadata)
    → Bedrock (Claude — extract entities, classify, summarize)
    → EventBridge (route to downstream: claims/underwriting/compliance)

Pattern: RAG Chatbot (Customer Support)
  Docs → S3 → Bedrock Knowledge Base → OpenSearch Serverless
  User → API GW → Lambda → Bedrock Agent (Claude) → RAG retrieval → Response
  History: DynamoDB (conversation state)
  Guardrails: Bedrock Guardrails (PII, topic filtering)

Interview Talking Points:
  ✅ Chunking strategy (fixed vs semantic vs hierarchical)
  ✅ Embedding model selection (Titan, Cohere)
  ✅ Reranking (Cohere Rerank, cross-encoder)
  ✅ Evaluation: RAGAS metrics (faithfulness, relevancy, context recall)
  ✅ Cost control: cache frequent queries in ElastiCache
```

### 🌐 IoT Architecture
```
Pattern: Industrial IoT (Manufacturing / Logistics)
  Devices
    → AWS IoT Core (MQTT broker; 2B+ messages/day)
    → IoT Rules Engine (SQL-like) →
        ├── Kinesis Data Streams → Lambda → DynamoDB (hot path)
        ├── S3 (cold path via Firehose)
        └── SNS (alerts)
    → IoT TwinMaker (digital twin)
    → Grafana (dashboards via Managed Grafana)

Key Services:
  IoT Core         → Device gateway + message broker
  IoT Greengrass   → Edge compute (Lambda on device)
  IoT Device Defender → Security: audit, detect anomalies
  IoT TwinMaker    → 3D digital twin connected to time-series data
  Timestream       → Purpose-built time-series DB (IoT metrics)

Security:
  - X.509 certs per device (provisioned via Fleet Provisioning)
  - Just-in-time provisioning (JITP) for large device fleets
  - IoT policies attached to certificates (not IAM users)
```

### 🛒 Retail Architecture
```
Pattern: E-commerce Platform
  Frontend: CloudFront → S3 (React/Next.js SPA)
  API:      API Gateway + Lambda (or EKS for complex services)
  Products: Aurora PostgreSQL (catalog, inventory)
  Cart:     ElastiCache Redis (session, cart; sub-ms)
  Orders:   DynamoDB (high write throughput, global tables)
  Search:   OpenSearch (product search, facets, typo-tolerance)
  Recommendations: Personalize (real-time, collaborative filtering)
  Events:   EventBridge → SQS → Lambda (order processing pipeline)
  Payments: Step Functions (orchestrate; compensating txn on failure)
  CDN:      CloudFront (static assets, API caching, WAF)

Scale Considerations:
  Black Friday traffic:
    → SQS buffer for order processing (absorb spikes)
    → Aurora Serverless v2 for auto-scale DB
    → Lambda + DynamoDB (infinite scale path)
    → Pre-warm provisioned Lambda concurrency before sales events

Interview Talking Points:
  ✅ Cache-aside pattern (ElastiCache + Aurora)
  ✅ Saga pattern for distributed transactions (Step Functions)
  ✅ CQRS: separate read (DynamoDB GSI/OpenSearch) from write path
  ✅ Idempotency: SQS deduplication + DynamoDB conditional writes
```

### 🏦 Banking / Financial Services Architecture
```
Pattern: Core Banking Modernization
  Legacy Core (on-prem) ←→ Direct Connect (redundant) ←→ AWS
  
  API Layer:
    API GW (private, VPC endpoint) → Lambda / EKS
  
  Data Layer:
    OLTP:     Aurora PostgreSQL (Multi-AZ, encrypted KMS CMK)
    Ledger:   QLDB (immutable, cryptographically verifiable)
    Cache:    ElastiCache Redis (account balance, session)
    Archive:  S3 Glacier (7-year regulatory retention, Object Lock WORM)
  
  Compliance:
    CloudTrail (all API calls) → S3 + CloudWatch Logs
    AWS Config  → Compliance rules (CIS Benchmark, NIST)
    Macie       → PII detection in S3
    SecurityHub → Aggregated security findings
    GuardDuty   → Threat detection (ML-based)
  
  Network Isolation:
    All workloads in private subnets
    PrivateLink for AWS service access (no public internet)
    WAF + Shield Advanced on public endpoints
    Network Firewall for stateful packet inspection

Compliance Frameworks:
  PCI DSS → Cardholder data environment isolation
  SOC 2   → AWS compliance reports via Artifact
  FIPS 140-2 → CloudHSM for key management
  OSFI (Canada) → Data residency in ca-central-1
```

### 🛡️ Insurance Architecture
```
Pattern: Claims Processing Automation
  Claim Intake:
    Web/Mobile → API GW → Lambda → SQS (claims queue)
  
  Document Processing:
    S3 (policy docs, claim photos)
    → Textract (extract forms, tables)
    → Comprehend (classify, entity extraction)
    → Bedrock (damage assessment summary, coverage recommendation)
  
  Rules Engine:
    DynamoDB (rules, coverage tables)
    → Lambda (eligibility evaluation)
    → Step Functions (multi-step approval workflow)
  
  Fraud Detection:
    Fraud Detector (ML model trained on historical claims)
    → High-risk → human review queue (SQS FIFO)
    → Low-risk  → auto-approve → payment API
  
  Reporting:
    Redshift (claims analytics, loss ratio, reserve calculations)
    QuickSight (actuarial dashboards)

Interview Talking Points:
  ✅ Step Functions for long-running claims workflows (up to 1 year)
  ✅ Human review loop (A2I — Augmented AI)
  ✅ Model explainability for regulatory compliance (SageMaker Clarify)
```

### 🚚 Logistics Architecture
```
Pattern: Real-time Fleet & Shipment Tracking
  GPS Devices
    → IoT Core (MQTT)
    → Kinesis Data Streams (position updates)
    → Lambda (geo-processing; Amazon Location Service)
    → DynamoDB (current position; TTL for old records)
    → EventBridge (geofence events → alerts → SQS → Lambda)
  
  Route Optimization:
    Amazon Location Service (maps, geocoding, routing)
    SageMaker (ETA prediction model)
  
  Warehouse Management:
    EKS / Lambda (microservices per domain)
    Aurora (inventory, orders, WMS)
    Kinesis Firehose → S3 → Glue → Redshift (analytics)
  
  Notifications:
    SNS → SES (email) / Pinpoint (SMS/push) → customer updates
    EventBridge → partner webhooks (carrier APIs)

Interview Talking Points:
  ✅ Amazon Location Service: geofencing, tracker, maps — avoid Google Maps cost
  ✅ DynamoDB TTL for position history (cost-efficient time-bounded data)
  ✅ Kinesis vs SQS: Kinesis for ordered, replayable stream; SQS for task queue
```

---

## 12. TROUBLESHOOTING MASTER GUIDE

### Performance Issues
```
HIGH LATENCY?
├── API / Web
│   ├── CloudWatch: P99 latency > P50? → tail latency issue → look at outlier instances
│   ├── X-Ray Service Map: which segment is slow?
│   ├── RDS/Aurora slow queries? → Performance Insights → Top SQL
│   └── Lambda cold starts? → Enable Provisioned Concurrency
│
├── Database
│   ├── CPU high? → Read Replicas + read traffic routing
│   ├── IOPS bottleneck? → upgrade EBS gp3 (3000 IOPS free); or io2
│   ├── Connection exhaustion? → RDS Proxy (connection pooling)
│   └── Cache miss rate? → ElastiCache; check cache eviction metric
│
└── Network
    ├── High RTT? → Check placement groups for EC2
    ├── NAT GW bandwidth? → scale out, or use VPC endpoints
    └── CloudFront cache hit rate low? → Check TTL, Cache-Control headers

COST SPIKE?
├── Cost Explorer → filter by service → identify anomaly date
├── Trusted Advisor → idle resources, unattached EBS, old snapshots
├── S3 → check PUT/GET requests (often APIs calling too frequently)
├── Data Transfer → NAT GW + inter-AZ transfer biggest surprises
└── Lambda → duration * invocations → identify hot functions
```

### Availability & Resilience
```
MULTI-AZ vs MULTI-REGION:
  Multi-AZ:     Same region; protects against AZ failure; RTO minutes
  Multi-Region: Different regions; protects against region failure; RTO seconds-minutes (active-active)

RTO / RPO Matrix:
  RPO=0, RTO=0 → Active-Active Multi-Region (Aurora Global + Route53 failover)
  RPO<1s, RTO<1min → Active-Passive (Aurora Global + automated failover)
  RPO<1hr, RTO<4hr → Warm standby (scaled-down replica in DR region)
  RPO<24hr, RTO<24hr → Backup/Restore (S3 cross-region replication + CloudFormation)

Resilience Patterns:
  Circuit Breaker  → Lambda + DynamoDB (track failure rate; open circuit)
  Retry + Backoff  → Exponential backoff with jitter (SDK default)
  Bulkhead         → Separate Lambda functions/EKS namespaces per domain
  Timeout          → Always set connection + read timeouts; never block indefinitely
  Graceful Degrade → Return cached/partial data when dependency down
  Chaos Engineering → AWS Fault Injection Simulator (FIS)
```

---

## 13. ARCHITECTURE STEP SEQUENCES (Whiteboard Scripts)

### "Design a Scalable Web App" (Classic Interview Question)
```
Step 1: Clarify requirements
  - Traffic: requests/sec, peak vs average?
  - Latency: P99 target?
  - Geography: single region or global?
  - State: stateless or sessions?

Step 2: Start simple → add complexity
  Route53 → CloudFront → ALB → EC2/ECS → Aurora (RDS) → ElastiCache

Step 3: Add scale
  Auto Scaling Group (EC2) / Fargate (ECS) → horizontal scale
  Aurora Read Replicas → read scaling
  ElastiCache Redis → cache-aside for reads

Step 4: Add resilience  
  Multi-AZ: ALB, ECS, Aurora (Multi-AZ or Global)
  S3 + CDN: static assets offloaded
  SQS buffer for async operations

Step 5: Add security
  WAF on CloudFront/ALB
  SGs: ALB→App, App→DB (no direct access)
  Secrets Manager for DB credentials
  KMS encryption at rest

Step 6: Add observability
  CloudWatch dashboards + alarms
  X-Ray distributed tracing
  Centralized logging → CloudWatch Logs Insights
```

### "Design for High Volume Data Migration" (AWS Specialty)
```
Step 1: Assess
  - Source: type (RDBMS/NoSQL/files), size, change rate
  - Network: bandwidth available to AWS
  - Downtime tolerance: zero, hours, days?
  - Compliance: data residency, encryption requirements

Step 2: Choose migration path
  > 10 TB + limited bandwidth → Snowball
  DB + near-zero downtime → DMS Full Load + CDC
  Files/NFS → DataSync
  VMs → AWS MGN

Step 3: Network setup
  Direct Connect (or VPN) for ongoing CDC replication
  Site-to-site VPN as backup path

Step 4: Execute
  Pre: schema conversion (SCT), test run on subset
  During: full load, then CDC; monitor DMS task health
  Validate: row counts, data checksums, app smoke tests
  Cutover: freeze source writes → final CDC catch-up → switch DNS/connection string

Step 5: Verify & decommission
  Run parallel for 48-72 hrs
  Monitor for errors, missing records
  Decommission source when confidence high
```

---

## 14. COST OPTIMIZATION QUICK REFERENCE

### Savings Levers
```
COMPUTE:
  Savings Plans (1 or 3 yr)  → up to 72% off On-Demand (flexible across EC2/Lambda/Fargate)
  Reserved Instances         → up to 75% off (specific instance type/AZ/region)
  Spot Instances             → up to 90% off; use for stateless, fault-tolerant, batch
  Graviton (ARM) instances   → 20-40% better price/performance (m7g, c7g, r7g)
  Right-sizing               → Compute Optimizer recommendations (weekly)

STORAGE:
  S3 Intelligent-Tiering     → Auto-move to cheaper tiers; no retrieval fees for IT
  EBS: gp2 → gp3             → 20% cheaper, 3x higher base IOPS
  EBS snapshot lifecycle     → Auto-delete old snapshots
  S3 Lifecycle rules         → IA after 30 days, Glacier after 90 days

NETWORKING:
  VPC Endpoints              → Eliminate NAT GW charges for S3/DynamoDB/etc.
  Same-AZ traffic            → Co-locate app + DB in same AZ to avoid inter-AZ transfer
  CloudFront                 → Reduce origin data transfer (origin shield)
  S3 Transfer Acceleration   → Only for uploads; test vs direct S3

DATABASE:
  Aurora Serverless v2       → Scale to zero (dev/test); right-size prod
  Reserved RDS               → 1-yr, 3-yr commitments for steady-state
  Redshift Serverless        → Burst workloads; pay per RPU second
```

---

## 15. INTERVIEW CHEAT SHEET — Top 20 AWS Questions

```
Q: What's the difference between SQS Standard and FIFO?
A: Standard: unlimited TPS, at-least-once delivery, best-effort ordering.
   FIFO: 300 TPS (3000 batched), exactly-once, strict ordering per MessageGroupId.

Q: How does Aurora differ from RDS Multi-AZ?
A: Aurora shares storage across AZs (6 copies, 3 AZs). Failover < 30s.
   RDS Multi-AZ = synchronous standby, failover 1-2 min, no performance benefit.

Q: When would you use EventBridge vs SNS vs SQS?
A: SNS = fan-out (1 → N); SQS = queue/decouple; EventBridge = routing by rules, SaaS, schema registry.

Q: How do you secure a microservices architecture on EKS?
A: IRSA (pod-level IAM), Secrets Manager CSI driver, Network Policies, service mesh mTLS, OPA/Kyverno for admission control, ECR image scanning.

Q: What's the difference between ALB and NLB?
A: ALB: L7 (HTTP/S), path/host routing, WAF integration, WebSocket.
   NLB: L4 (TCP/UDP), ultra-low latency, static IPs, PrivateLink.

Q: How would you migrate a 50 TB Oracle database to Aurora PostgreSQL with minimal downtime?
A: SCT for schema conversion → DMS Full Load → DMS CDC replication → validate → cutover.

Q: Explain DynamoDB GSI vs LSI.
A: LSI: alternate sort key, same partition key, created at table creation only, shares table RCU/WCU.
   GSI: entirely new key schema, can be added after, own RCU/WCU.

Q: How do you achieve RPO=0 on AWS?
A: Aurora Global DB (< 1s replication) + Route53 health check + ALB in secondary region. Active-Active = RPO 0.

Q: What is AWS PrivateLink and when do you use it?
A: Expose a service (NLB) privately to VPCs without VPC peering or internet. Used for SaaS services, sharing internal services, keeping traffic in AWS network.

Q: How would you design for 1 million requests/second?
A: CloudFront (edge cache) → API GW (throttle) → Lambda (auto-scale) → DynamoDB (on-demand). Or: NLB → EKS (HPA+Karpenter) → Redis → Aurora.

Q: What is Kinesis Firehose vs Kinesis Data Streams?
A: Data Streams = real-time, ordered, replay, consumers pull. Firehose = fully managed delivery to S3/Redshift/OpenSearch; no coding; built-in transformation.

Q: Explain Transit Gateway vs VPC Peering.
A: Peering: 1:1, no transitive routing, no bandwidth limit. TGW: hub-spoke, thousands of VPCs, transitive routing, $0.05/GB attachment.

Q: How do you handle secrets in a containerized app on EKS?
A: Secrets Manager + CSI Secrets Store Driver (mounts as file/env var) + IRSA (pod assumes IAM role to access secret). Never put secrets in image or env vars directly.

Q: What is the difference between CloudWatch Logs and CloudTrail?
A: CloudWatch Logs = application/system logs, custom metrics. CloudTrail = AWS API call audit trail (who did what, when, from where).

Q: How would you implement a cost alert?
A: AWS Budgets (threshold alert) → SNS → email/Slack. Or Cost Anomaly Detection (ML-based, catches sudden spikes).

Q: What is SageMaker Model Monitor?
A: Monitors deployed models for data drift, model quality drift, bias drift. Triggers CloudWatch alarms for retraining.

Q: Explain Lambda cold starts and how to fix them.
A: First invocation = container init (runtime + code load). Fix: Provisioned Concurrency (keeps N containers warm), Lambda SnapStart (Java), slim package, avoid VPC unless needed.

Q: What is the difference between NAT Gateway and NAT Instance?
A: NAT GW = managed by AWS, scales to 100 Gbps, no security group, AZ-specific (deploy in each AZ for HA). NAT Instance = EC2, you manage, can be SG-controlled, cheaper at low scale.

Q: How does IAM evaluation work when multiple policies apply?
A: Explicit DENY wins always → check SCP → Resource Policy → Permission Boundary → Identity Policy. Default is implicit deny.

Q: What is AWS Organizations SCP and how does it differ from IAM?
A: SCP sets maximum permissions for all accounts in an OU — it doesn't grant permissions. IAM grants permissions within those boundaries. SCP can't be overridden by account-level IAM admin.
```

---

## 17. THINKING FRAMEWORK — Never Default to AWS-Only

> This section exists because defaulting to AWS managed services is a blind spot.
> Real-world projects — POC (Proof of Concept) and enterprise — use BOTH AWS native
> services AND open-source tools. Always think in two layers before answering.

### The Two-Layer Rule
```
LAYER 1 — OPEN SOURCE (OSS) STACK
  Ask: "What OSS tool is the industry standard for this problem?"
  LangChain / LangGraph  → LLM orchestration, agentic workflows
  LlamaIndex             → RAG (Retrieval-Augmented Generation) data ingestion
  LangSmith              → LLM observability, tracing, evaluation
  MLflow                 → ML experiment tracking, model registry
  Apache Airflow / MWAA  → Workflow orchestration, data pipeline scheduling
  dbt (data build tool)  → SQL-based data transformation in warehouse
  Apache Kafka           → High-throughput event streaming
  Hugging Face           → Pre-trained models, fine-tuning, embeddings
  RAGAS                  → RAG pipeline evaluation metrics

LAYER 2 — AWS DEPLOYMENT
  Ask: "Where and how does this OSS tool run on AWS?"
  LangGraph app    → ECS (Elastic Container Service) Fargate or EC2
  MLflow server    → EC2 + RDS (metadata) + S3 (artifacts)
  Airflow          → Amazon MWAA (Managed Workflows for Apache Airflow)
  Kafka            → Amazon MSK (Managed Streaming for Apache Kafka)
  HF models        → SageMaker Endpoints or EC2 GPU instances
  dbt              → runs against Redshift, Athena, or Aurora
```

### When to Pick OSS vs AWS Native — Quick Guide
| Problem | OSS Choice | AWS Native Choice | Pick OSS When | Pick Native When |
|---|---|---|---|---|
| LLM agent orchestration | LangGraph | Bedrock Agents | Complex branching, cycles, full control | Simple tool use, zero infra management |
| RAG ingestion | LlamaIndex | Bedrock Knowledge Bases | Custom chunking, 10K+ docs, multi-index | Quick POC, no custom logic needed |
| LLM observability | LangSmith | CloudWatch + X-Ray | Prompt debugging, RAG eval, dev speed | Regulated env, must stay in AWS |
| Experiment tracking | MLflow | SageMaker Experiments | Multi-cloud, portability, open ecosystem | AWS-only shop, SageMaker pipeline |
| Pipeline scheduling | Airflow | Step Functions | Complex DAGs, data engineering teams | Event-driven, serverless, simple flows |
| Data transformation | dbt | Glue ETL | SQL-first teams, version control, lineage | Spark jobs, non-SQL transformations |
| Event streaming | Kafka (MSK) | Kinesis | Existing Kafka ecosystem, KSQL | Greenfield AWS, fully managed preferred |
| Embeddings | sentence-transformers | Bedrock Embeddings | Domain fine-tuning, cost at scale | Managed, no GPU management wanted |

### POC vs Enterprise Stack Reality
```
POC (Proof of Concept) — "Show it works in 1-2 weeks"
  Single EC2 instance (g4dn.xlarge for GPU workloads)
  LangChain + LlamaIndex + LangSmith
  S3 + OpenSearch Serverless
  Bedrock (Claude) for LLM
  SQLite or DynamoDB for metadata
  Cost: ~$50-200/month

ENTERPRISE PRODUCTION — "Scale, govern, operate"
  EKS (Elastic Kubernetes Service) or ECS for containerized services
  LangGraph (agents) + LlamaIndex (RAG) + LangSmith (observability)
  S3 Data Lake (Bronze/Silver/Gold) + Glue + Lake Formation
  Aurora + Redshift + OpenSearch Serverless
  Bedrock + fine-tuned HuggingFace models on SageMaker
  MLflow on EC2 + Airflow on MWAA for pipelines
  Bedrock Guardrails + SageMaker Clarify + CloudTrail for governance
  Cost: $5,000-50,000+/month depending on scale
```

---

## 16. UPDATE LOG

```
[INITIAL] — Created: 2026-05-20
Sections: Compute, Networking, Storage, Databases, Messaging,
          Security, Observability, AI/ML, Data Platform, Migration,
          Use Cases (AI, IoT, Retail, Banking, Insurance, Logistics),
          Troubleshooting, Whiteboard Scripts, Cost Optimization,
          Interview Cheat Sheet (Top 20 Q&A)

[TODO — Add Next]
□ EKS Deep Dive (Karpenter, KEDA, Cilium CNI)
□ Serverless Architecture Patterns (event-driven, fan-out, saga)
□ Disaster Recovery Runbooks per use case
□ CDK / CloudFormation / Terraform on AWS comparison
□ Well-Architected Framework pillars mapped to services
□ AWS Service Quotas & Limits (common interview gotchas)
□ Multi-Account Strategy (Landing Zone, Control Tower)
□ Networking: BGP, ECMP, Direct Connect Gateway deep dive
□ Cost deep-dive per use case (Banking, Retail, AI)
□ SageMaker MLOps pipeline end-to-end
```

---

*End of AWS_Architecture_MindMap.md — Keep appending, keep winning.*
