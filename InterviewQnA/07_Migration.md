# AWS Interview Q&A — Migration Strategies & Execution

> **The Migration Story:** Moving a company's digital infrastructure to the cloud is like
> performing heart surgery while the patient is still running a marathon. The business can't
> stop. Transactions must process. Customers must be served. And somewhere in the middle of
> all that, you're replacing the entire foundation. This section covers how to do it without
> killing the patient.

---

## Table of Contents
1. [Migration Strategy — The 7 R's](#7rs)
2. [Application Migration — AWS MGN](#mgn)
3. [Database Migration — DMS & SCT](#dms)
4. [File & Data Migration — DataSync & Snowball](#datasync)
5. [Zero-Downtime Cutover Patterns](#cutover)
6. [Migration Anti-Patterns & War Stories](#antipatterns)

---

## 1. Migration Strategy — The 7 R's {#7rs}

---

### Q1: What are the 7 R's of migration and when do you apply each?

**The Story:**
In 2011, Gartner published "5 R's of Cloud Migration." AWS expanded it to 6, then 7.
The decision of which "R" to apply to each workload is the most important judgment call in any migration.
Get it wrong and you've wasted millions.

```
R1 — RETIRE (easiest, often overlooked)
  "We just discovered 30% of our apps haven't been used in 2 years."
  Action: Decommission. Don't migrate what doesn't need to exist.
  How to find: AWS Migration Hub, application usage metrics, stakeholder interviews
  Saves: migration effort, cloud costs, maintenance burden
  Reality: Most orgs find 20-30% of apps can be retired in discovery phase

R2 — RETAIN (strategic decision)
  "This mainframe runs the general ledger. We can't risk touching it this year."
  Action: Keep on-premises. Come back to it in 2 years.
  When to retain:
    → Compliance requires on-prem (some financial regulators)
    → App recently modernized on-prem (sunk cost + re-migration risk)
    → Technical dependency on physical hardware (robotics, specialized equipment)
    → App depends on ultra-low latency from physical colocation

R3 — REHOST ("Lift and Shift")
  "Pick the server up and put it in AWS. Same OS, same code, nothing changes."
  Tool: AWS Application Migration Service (MGN)
  Time: Days to weeks per application
  Risk: Low (nothing changes)
  Savings: Immediate: right-size instances, Savings Plans, eliminate data center costs
  Long-term savings: Limited (you're still running legacy arch on cloud)
  Use when: Fast migration required, large portfolio, tech modernization comes later

R4 — REPLATFORM ("Lift, Tinker, and Shift")
  "Move to AWS, but swap MySQL for RDS. Or move from Tomcat to Elastic Beanstalk."
  Keep core architecture, replace one component with a managed service.
  Examples:
    → Oracle on EC2 → Oracle on RDS (get managed backups, Multi-AZ)
    → App server on EC2 → App on Elastic Beanstalk (managed deployment)
    → Self-managed Kafka → Amazon MSK (managed Kafka)
  Risk: Medium (one component changed)
  Savings: Better than rehost — managed services reduce ops overhead

R5 — REPURCHASE ("Drop and Shop")
  "Replace our on-prem CRM (Siebel) with Salesforce."
  Action: Move to a SaaS product.
  Examples:
    → Exchange Server → Microsoft 365
    → SAP on-prem → SAP on AWS or move to cloud ERP
    → HR system → Workday
  Risk: Business process change (training, data migration, integration)
  Savings: No infrastructure to manage at all

R6 — REFACTOR / RE-ARCHITECT
  "Tear it down and rebuild it as microservices/serverless."
  This is the most valuable — and most expensive/risky option.
  When justified:
    → Current app literally cannot scale (monolith hitting walls at peak)
    → Business needs new features the old arch cannot support
    → Security and compliance gaps too deep to patch
  Examples:
    → Monolith → microservices on EKS
    → Batch overnight processing → real-time Lambda event processing
    → On-prem file storage → S3 + Lambda + DynamoDB
  Cost: High (rewrite effort)
  Value: Highest long-term (cloud-native capabilities, auto-scale, pay-per-use)

R7 — RELOCATE ("VMware to VMware Cloud on AWS")
  "Move VMware workloads to AWS without changing anything — not even the VMs."
  Tool: VMware Cloud on AWS
  Use when: Deep VMware investment, operators don't want to retrain, large VM fleets
  What changes: Physical location (your DC → AWS DC), billing model
  What doesn't change: vSphere management, VM images, network policies
```

---

### Q2: How do you prioritize which applications to migrate first?

**The Story:**
A migration program covering 500 applications cannot start with the most critical payment
processing system. And it shouldn't start with the most complex application either.

```
MIGRATION WAVE PLANNING:

WAVE 0 — FOUNDATION (before any workloads):
  → Landing Zone (Control Tower)
  → Network (Transit Gateway, Direct Connect)
  → Security tooling (GuardDuty, SecurityHub)
  → Monitoring (CloudWatch, centralized logging)
  → Identity (IAM Identity Center, AD integration)
  Duration: 4-8 weeks
  Why first: Every subsequent wave depends on this infrastructure

WAVE 1 — PILOTS (Low-risk, high-learning):
  → Non-critical internal tools (timekeeping, project management)
  → Dev/test environments for active projects
  → Applications with no external dependencies
  Duration: 4-8 weeks
  Why first: Build team skills, prove the process, find surprises with low blast radius

WAVE 2 — QUICK WINS:
  → Applications that scored well on migration readiness assessment
  → Simple web apps, stateless services
  → Apps where owner is enthusiastic and cooperative
  Duration: 8-16 weeks

WAVE 3+ — COMPLEX WORKLOADS:
  → Database-heavy applications (DMS migrations)
  → Tightly coupled systems requiring coordinated cutover
  → Applications requiring refactoring

LAST — CRITICAL SYSTEMS:
  → Core banking, payment processing, ERP
  → After team has run 50+ successful migrations
  → Multiple runbook rehearsals, DR in place

PRIORITIZATION MATRIX:
  Score each app on:
  Business Criticality (1-5)  × Migration Complexity (1-5)
  
  Low criticality + low complexity → first waves (easy wins)
  High criticality + low complexity → early waves (high value, manageable risk)
  Low criticality + high complexity → later (complex but not urgent)
  High criticality + high complexity → last (de-risk everything first)
```

---

## 2. Application Migration — AWS MGN {#mgn}

---

### Q3: How does AWS Application Migration Service (MGN) work?

**The Story:**
AWS MGN (formerly CloudEndure) solves the "how do I pick up a running server and move it to AWS"
problem. The key challenge: production servers are live and changing every second. You can't just
take a snapshot and restore it — by the time the restore is done, the data is hours out of date.

MGN uses continuous replication:

```
STEP 1: INSTALL AGENT
  → Install AWS Replication Agent on source server (Windows or Linux)
  → Agent authenticates to AWS using IAM credentials
  → No reboot required, no downtime during installation

STEP 2: INITIAL REPLICATION
  → Agent reads all source disk data (live, while server is running)
  → Sends to a Replication Server in AWS (small EC2 instance in your VPC)
  → Replication Server writes to EBS volumes → staging area
  → First sync: may take hours to days for large disks

STEP 3: CONTINUOUS REPLICATION
  → After initial sync, agent tracks block-level changes (like CDC for disks)
  → Every write on the source is replicated to AWS within seconds
  → Lag typically < 1 second
  → Source server continues running — no impact on production

STEP 4: TEST MIGRATION
  → Launch a test instance in AWS from current replica
  → Test: does the application start? Can it connect to DB? Does the app function?
  → Test instances don't affect the replication pipeline
  → Test as many times as you want before the real cutover

STEP 5: CUTOVER
  → Schedule: initiate cutover during maintenance window
  → MGN waits for replication lag = 0 (fully caught up)
  → Launches final production instance from replica
  → Update DNS/load balancer to point to AWS instance
  → Source server decommissioned after validation period

TYPICAL TIMELINE:
  Install agent:        15 minutes
  Initial replication:  2-48 hours (depends on disk size, bandwidth)
  Testing:              As long as needed
  Final cutover:        30-60 minutes (DNS change + validation)
  Total downtime:       ~30 minutes (just for final cutover)
```

---

## 3. Database Migration — DMS & SCT {#dms}

---

### Q4: Walk me through migrating a 10TB Oracle database to Aurora PostgreSQL with near-zero downtime.

**The Story:**
This is the Everest of database migrations. Oracle is complex: stored procedures, PL/SQL,
sequences, custom data types, triggers, Oracle-specific functions. Aurora PostgreSQL speaks
a different dialect. And the database is live — orders are being placed while you migrate.

```
PHASE 1 — ASSESSMENT (1-2 weeks)
  AWS Schema Conversion Tool (SCT):
    → Connect to source Oracle
    → Analyze all objects: tables, views, procedures, functions, triggers, indexes
    → Generate conversion report:
       ✅ Automatically converted: 70-80% of objects
       ⚠️  Needs review: 10-15% (minor dialect differences)
       ❌  Manual conversion: 5-10% (Oracle-specific features, no PG equivalent)
    → The manual conversion items are your timeline risk — assess carefully

  Key Oracle → Aurora PostgreSQL conversion challenges:
    PL/SQL → PL/pgSQL (similar, but different)
    ROWNUM → ROW_NUMBER() OVER() or LIMIT/OFFSET
    SYSDATE → CURRENT_TIMESTAMP
    Oracle sequences → PostgreSQL sequences (different syntax)
    CONNECT BY (hierarchical queries) → Recursive CTEs
    CLOB/BLOB → TEXT/BYTEA
    Oracle packages → separate PostgreSQL functions/procedures

PHASE 2 — SCHEMA CONVERSION (2-4 weeks)
  → SCT converts schema → review and fix flagged items
  → Create target Aurora PostgreSQL cluster
  → Apply converted schema
  → Test: can the application even start against the new schema?

PHASE 3 — FULL LOAD (1-3 days depending on size)
  AWS DMS (Database Migration Service):
    → Create replication instance (size matters: at least r5.2xlarge for 10TB)
    → Create source endpoint (Oracle)
    → Create target endpoint (Aurora PostgreSQL)
    → Create migration task: Full Load mode
    → DMS reads entire source, writes to target
    → Duration: depends on bandwidth and instance size
    → Monitor: table statistics in DMS console

PHASE 4 — CDC (Change Data Capture) — runs while business is live
  → After full load completes, switch task to CDC mode
  → DMS reads Oracle redo logs → captures every INSERT/UPDATE/DELETE
  → Applies changes to Aurora PostgreSQL in near real-time
  → Check: replication lag metric (should be < 60 seconds for most workloads)
  → Run for days/weeks to build confidence

PHASE 5 — VALIDATION (run parallel)
  AWS DMS Data Validation:
    → Row counts match per table
    → Checksum validation on random samples
  Application-level testing:
    → Run your test suite against Aurora PostgreSQL
    → Compare query results between Oracle and Aurora (spot check 50 key queries)
    → Performance testing: some Oracle queries may need index adjustments in PG

PHASE 6 — CUTOVER (the critical window)
  Pre-cutover:
    → Prepare new connection strings, Secrets Manager entries
    → Verify CDC lag is < 5 seconds
    → Brief maintenance window scheduled (even 15 minutes is enough)
  
  Cutover execution:
    T+0:   Set application to maintenance mode (or stop accepting writes)
    T+1:   Confirm CDC lag reaches 0 (all changes replicated)
    T+2:   Point application at Aurora PostgreSQL connection string
    T+5:   Smoke test: place a test order, verify it appears in Aurora
    T+10:  Remove maintenance mode → production traffic to Aurora PostgreSQL
    T+15:  Monitor: error rates, latency, query performance
    T+60:  If healthy, declare success; keep Oracle running for 48-72h as safety net

ROLLBACK PLAN (must exist before you start):
  If Aurora has errors:
    → Switch connection string back to Oracle (2 minutes)
    → DMS CDC is still running (can resume when you try again)
    → Root cause, fix, retry next window
```

---

### Q5: What are the most common DMS migration failures?

```
FAILURE 1 — LOB (Large Object) columns
  Problem: Oracle CLOBs/BLOBs don't transfer like regular columns
  Solution: Enable "LOB settings" in DMS task → "Full LOB mode" (slow but complete)
            Or "Limited LOB mode" with size cap (faster, but truncates if > limit)

FAILURE 2 — Character encoding mismatch
  Problem: Oracle using WE8ISO8859P1, Aurora expecting UTF-8
           Special characters get mangled or cause errors
  Solution: Set character set in DMS endpoints BEFORE running full load
            Test with rows containing: accents, Japanese characters, emoji

FAILURE 3 — Sequence/identity column drift
  Problem: Oracle sequences have current value; DMS doesn't migrate the counter
           Aurora sequences start at 1, conflict with existing PKs
  Solution: After full load, manually set sequences to max(id)+1000 in Aurora
            SQL: SELECT setval('my_sequence', (SELECT max(id)+1000 FROM my_table));

FAILURE 4 — Stored procedure behavior differences
  Problem: Application calls Oracle procedure → PG version has subtle bug
           Unit tests pass; production edge case fails
  Solution: SCT automated tests + manual testing of ALL stored procedures
            Critical: test with PRODUCTION DATA COPY (not synthetic test data)

FAILURE 5 — Index on reserved keyword column name
  Problem: Oracle allows column named "order". PostgreSQL requires quoting: "order"
           App SQL without quotes fails on PostgreSQL
  Solution: SCT catches most; audit any raw SQL in application code

FAILURE 6 — Network bandwidth bottleneck
  Problem: 10TB full load at 100 Mbps = 22 hours; Oracle redo logs pile up
           CDC falls behind → migration stalls
  Solution: Size the DMS replication instance correctly (more vCPUs = more throughput)
            Use Direct Connect for migration traffic if WAN is the bottleneck
            Run during low-traffic period for full load
```

---

## 4. File & Data Migration — DataSync & Snowball {#datasync}

---

### Q6: When do you use DataSync vs Snowball vs Direct Copy to S3?

**The Story:**
You have 500TB of files on a NAS (Network Attached Storage) in your data center.
Your internet pipe is 1 Gbps. How do you get this data to S3?

```
DECISION MATH:

Data Size: 500 TB
Internet Speed: 1 Gbps = 125 MB/s
Time to transfer (raw): 500,000 GB ÷ 125 MB/s ÷ 3600 = ~1,111 hours = 46 days

Is 46 days acceptable? Probably not.
Can you get a dedicated 10 Gbps Direct Connect? Maybe, but takes 60-90 days to provision.

SNOWBALL EDGE STORAGE OPTIMIZED:
  Capacity: 80 TB usable per device
  Devices needed: 500 TB ÷ 80 TB = 7 devices (order in parallel)
  Transfer to device (10 GbE onsite): 80 TB ÷ 1.2 GB/s ≈ 18 hours per device
  Shipping (US): 2-3 days each way
  AWS ingestion: 3-5 days per device
  Total timeline: ~2 weeks for all 7 devices (parallel processing)
  Cost: $300/device + $0.03/GB transferred ≈ $17,100 for 500TB
  
  → Use Snowball when: > 10 TB, limited bandwidth, time is critical

DATASYNC (over existing network):
  What it does:
    → Installs DataSync Agent as a VM in your data center
    → Agent reads NFS/SMB shares directly
    → Transfers to S3/EFS/FSx using parallel threads with encryption
    → Verifies every file with checksums
    → Throttle bandwidth to not saturate WAN
    → Schedule transfers during off-peak hours
  
  Speed: Up to 10 Gbps (agent can use multiple threads)
  Best for:
    → Ongoing/incremental sync (not just one-time)
    → < 50 TB over reasonable bandwidth
    → Regular sync from on-prem to S3 (daily backups, DR replication)

HYBRID APPROACH (for 500TB scenario):
  Week 1-2: Snowball ships to move initial 500 TB snapshot to S3
  Day 1+: DataSync continuously syncs changes (delta only) over existing WAN
  Result: Near-real-time S3 mirror with low ongoing bandwidth cost
```

---

## 5. Zero-Downtime Cutover Patterns {#cutover}

---

### Q7: Explain the Strangler Fig pattern for migrating a monolith to microservices.

**The Story:**
Named after the strangler fig tree, which grows around an existing tree, taking over gradually
until the original is gone. This is the safest way to modernize a running application without
the "big rewrite" risk.

```
THE BIG REWRITE RISK:
  "Let's rewrite everything in 12 months, then switch."
  Month 12: New system is 80% done.
  Month 18: Still finding edge cases the old system handled.
  Month 24: Two systems to maintain; business demands new features in BOTH.
  This is called the Second System Effect. It kills projects.

STRANGLER FIG — INCREMENTAL REPLACEMENT:

Original monolith: [User] → [Monolith handles everything]

Step 1: Put a proxy/gateway in front (no behavior change yet)
  [User] → [API Gateway / ALB] → [Monolith]

Step 2: Extract one service (the "least risky" one first: e.g., email sending)
  [User] → [API Gateway]
              → POST /send-email → [New Email Microservice]  ← new!
              → Everything else → [Monolith]               ← unchanged

Step 3: Extract next service (product catalog)
  [User] → [API Gateway]
              → /send-email → [Email Service]
              → /products/* → [Product Catalog Service]    ← new!
              → Everything else → [Monolith]

...repeat over 12-24 months...

Final state:
  [User] → [API Gateway]
              → /email     → [Email Service]
              → /products  → [Product Service]
              → /orders    → [Order Service]
              → /payments  → [Payment Service]
  The original monolith is gone (strangled by the new services)

AWS IMPLEMENTATION:
  Proxy: Application Load Balancer with path-based routing
  New services: ECS Fargate microservices or Lambda functions
  Data: Strangler Sidecar pattern — services share DB initially, 
        then each service gets its own DB over time
  Risk: Near zero at each step (one small change at a time)
```

---

### Q8: How do you do a database cutover with zero downtime?

```
THE BLUE/GREEN DATABASE PATTERN:

Setup:
  BLUE (current production):     Oracle on-prem
  GREEN (target):                Aurora PostgreSQL on AWS
  DMS CDC running continuously between Blue and Green

Timeline:
  Days 1-14: DMS full load + CDC running
             Green catches up to Blue
             Green lag: consistently < 30 seconds
  
  Days 7-13: Application testing against Green
             Fix any query behavior differences
             Performance tuning (missing indexes, query rewrites)
  
  Day 14 (cutover night):
    T-30 min:  Enable database transaction counter monitoring
               "We must verify 0 lag before flipping"
    
    T-0:       Maintenance mode ON (or read-only mode if possible)
               Freeze application writes
    
    T+1 min:   Wait for DMS CDC lag = 0
               (All transactions that happened in Blue are now in Green)
    
    T+2 min:   Update connection string in Secrets Manager:
               Old: "host=oracle.internal.corp.com"
               New: "host=aurora-cluster.us-east-1.rds.amazonaws.com"
    
    T+3 min:   Application reads new secret → points to Green (Aurora)
    
    T+5 min:   Maintenance mode OFF
               First real transactions flowing through Aurora PostgreSQL
    
    T+15 min:  Smoke test: place test order, verify in database
               Check: response time, error rates, query plans
    
    T+60 min:  Healthy! 
               Keep Blue (Oracle) warm for 72 hours — emergency rollback available
    
    T+72 hrs:  Decommission Blue. Migration complete.

TOTAL APPLICATION DOWNTIME: 3-5 minutes
```

---

## 6. Migration Anti-Patterns {#antipatterns}

---

### Q9: What are the most common migration mistakes you've seen / would warn against?

```
MISTAKE 1 — "Lift and Shift Everything, Modernize Later"
  Reality: "Later" never comes. Once the app is in AWS and "working,"
           there's no business pressure to modernize further.
  Better: Group apps into: Rehost (this year), Replatform (next year), 
          Refactor (18-24 months) — with actual roadmap commitments.

MISTAKE 2 — Migrating Before the Network is Ready
  Company started migrating workloads to AWS before the Direct Connect was set up.
  All 50 EC2 instances are talking back to on-prem Oracle over a 100 Mbps VPN.
  Performance: terrible. Rollback required.
  Better: Wave 0 must include network connectivity. Nothing moves until DX is up.

MISTAKE 3 — Using the Default VPC
  Developer migrates an app to the default VPC (10.0.0.0/16).
  6 months later, you need to peer with another VPC. CIDR conflict — they both use 10.0.0.0/16.
  No way to peer overlapping CIDRs.
  Better: Design and allocate non-overlapping CIDRs for ALL accounts and environments upfront.

MISTAKE 4 — Not Testing the Rollback
  Every migration plan has a rollback section. Almost nobody tests it.
  Production cutover goes wrong → scramble to roll back → DMS reverse task not configured.
  Better: Do a mock cutover to GREEN, then deliberately roll back to BLUE. Test it.

MISTAKE 5 — Skipping the Data Validation Step
  DMS migrated 10 million rows. Team declares success.
  Two weeks later: finance report shows totals are off by $2 million.
  Root cause: DMS silently failed on 10,000 rows due to encoding issue.
  Better: Always run DMS Data Validation. Compare row counts AND checksums.

MISTAKE 6 — Cost Surprise After Migration
  Team rehosted 200 servers. Bill is higher than on-prem.
  They migrated like-for-like: same CPU, same RAM. On-prem servers were 15% utilized.
  On AWS, you pay for what you provision, not what you use (for EC2 On-Demand).
  Better: Right-size BEFORE or immediately after migration. 
          Use AWS Compute Optimizer. Move to Savings Plans/Spot where possible.
```

---

## Migration Quick Reference

```
DMS TASK TYPES:
  Full Load Only:              One-time migration, acceptable downtime during cutover
  Full Load + CDC:             Zero-downtime migration (start CDC immediately after full load)
  CDC Only:                    Already migrated schema/data manually; just sync changes

DMS REPLICATION INSTANCE SIZING:
  < 1 TB source:    r5.large or r5.xlarge
  1-10 TB:          r5.2xlarge or r5.4xlarge
  > 10 TB:          r5.8xlarge; consider parallel tasks per table group

KEY DMS METRICS TO MONITOR:
  CDCLatencySource:    How far behind source DB events? (target: < 30 sec)
  CDCLatencyTarget:    How far behind applying to target? (target: < 30 sec)
  FreeableMemory:      If low, DMS is struggling — upsize instance
  SwapUsage:           Any swap = underpowered; task will slow

MIGRATION TOOL CHEAT SHEET:
  VMs / Servers:        AWS MGN (Application Migration Service)
  Databases:            AWS DMS + SCT
  Files (NFS/SMB):      AWS DataSync
  Large volumes (> 10TB, limited bandwidth): Snowball Edge
  Huge (> 100 PB):      AWS Snowmobile (truck)
  Containers (K8s):     Helm + GitOps; re-deploy, don't "migrate"
  SaaS (VMware):        VMware Cloud on AWS
```

---

*Next: [08_Scaling_DR_HA.md](./08_Scaling_DR_HA.md) — Scaling, High Availability, and Resilience Engineering*
