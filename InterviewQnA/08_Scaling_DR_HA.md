# AWS Interview Q&A — Scaling, High Availability & Resilience

> **The Resilience Story:** Murphy's Law was written for distributed systems.
> Servers will fail. Network will partition. Databases will go offline. AZs will degrade.
> Regions have failed. The question isn't *if* something breaks — it's *what your system
> does when it does*. High Availability is engineering the answer to that question before
> the question is asked at 2am on a Friday.

---

## Table of Contents
1. [Auto Scaling Deep Dive](#autoscaling)
2. [High Availability Patterns](#ha)
3. [Disaster Recovery — Tiers & Runbooks](#dr)
4. [Database Resilience Patterns](#dbresil)
5. [Chaos Engineering](#chaos)
6. [Load Balancing Deep Dive](#lb)
7. [Scripting for Scaling & Automation](#scripting)

---

## 1. Auto Scaling Deep Dive {#autoscaling}

---

### Q1: Explain EC2 Auto Scaling groups — how do they maintain desired capacity?

**The Story:**
An Auto Scaling Group (ASG) is like a self-healing flock of servers. You define the rules.
The ASG enforces them continuously — adding capacity when the load grows, removing it when
it shrinks, and replacing unhealthy instances automatically.

```
ASG COMPONENTS:

Launch Template (what to launch):
  → AMI: which OS/image
  → Instance type: t3.medium or mixed (multiple types for Spot)
  → Security groups, key pair, IAM instance profile
  → User data: bootstrap script (install app, configure agent)
  → EBS volumes: size, type, encryption

ASG Configuration (how many to run):
  → Minimum: never fewer than N instances
  → Desired: target number right now
  → Maximum: never more than N instances

Health Checks (when to replace):
  → EC2 health check: is the instance running? (basic)
  → ELB health check: is the instance passing the load balancer health check? (better)
    If ALB health check fails: ASG terminates and replaces the instance
  → Custom health checks: your app can mark instance unhealthy via API

LIFECYCLE HOOKS (do things during launch/terminate):
  When launching:
    ASG launches instance → sends to "Pending:Wait" state
    → Your Lambda runs: register in service mesh, warm up cache
    → Trigger "CONTINUE" → instance moves to InService
  
  When terminating:
    ASG wants to terminate instance → sends to "Terminating:Wait"
    → Your Lambda runs: drain connections, backup state, deregister from monitoring
    → Trigger "CONTINUE" → instance terminates
    → Default timeout: 3600 seconds (1 hour)

TERMINATION POLICY (which instance to kill when scaling in):
  Default: 
    1. AZ with most instances (balance AZs first)
    2. Oldest launch template (kill old configs)
    3. Closest to billing hour (save money)
  Custom: OldestInstance, NewestInstance, ClosestToNextInstanceHour
```

---

### Q2: How do you scale a stateful application?

**The Story:**
Stateless apps (your API returns the same result regardless of which server answers) are easy to scale.
Add more servers. Any server can handle any request.

Stateful apps (the server knows something about you — your cart, your session, your partial upload)
are trickier. If the server holding your state goes down or you're routed to a different server, your
state is lost.

**The Solutions:**

```
SOLUTION 1 — EXTERNALIZE STATE
  Don't store state in memory on the server. Store it outside.
  
  Sessions → ElastiCache Redis
    User logs in → session stored in Redis with TTL
    Any app server reads session from Redis → stateless again
  
  File uploads → S3
    User uploads 500 MB video file → multipart upload directly to S3
    App server just coordinates, never holds the file
  
  WebSocket connections → ECS/EKS with sticky sessions OR API Gateway WebSocket
    Sticky sessions: ALB stickiness cookie routes same user to same server
    Watch out: if that server fails, the connection is lost anyway

SOLUTION 2 — CONSISTENT HASHING (for cache clusters)
  ElastiCache Redis cluster mode: distributes keys across nodes
  Each key has one primary node (+ replicas)
  When a node is added/removed: only keys on that node need to move
  Used for: session stores, real-time leaderboards, distributed locks

SOLUTION 3 — DATABASE-BASED STATE
  For long-running workflows:
    Step Functions → state stored in SFN service (not in your app)
    DynamoDB → workflow state persisted, any server can resume it
  
  This is how you build "resumable" processes:
  Order processing → Step Function task state → if Lambda times out, Step Functions retries
```

---

## 2. High Availability Patterns {#ha}

---

### Q3: Design a highly available three-tier web application on AWS.

**The Full Architecture:**

```
GLOBAL LAYER:
  Route 53 (latency-based routing with health checks)
  → Automatic failover to secondary region if primary health check fails
  
  CloudFront CDN
  → Cache static assets at 400+ edge locations
  → Reduce origin load by 70-90%
  → WAF rules at the edge

PRESENTATION TIER (high availability):
  Application Load Balancer
  → Multi-AZ: ALB spans all 3 AZs in the region
  → Health checks: HTTP /health endpoint every 30 seconds
  → If instance fails health check: ALB stops routing to it immediately
  → Connection draining: waits 30s for in-flight requests before removing instance

  Auto Scaling Group (EC2 or ECS Fargate):
  → Desired: 6 instances (2 per AZ)
  → Min: 3 (at least 1 per AZ — survive an AZ failure)
  → Max: 60 (scale for peak)
  → Health check grace period: 300s (app needs time to start)
  → Target tracking: CPU = 60% target

APPLICATION TIER:
  ECS Fargate or EKS:
  → Services spread across 3 AZs (topology spread constraints)
  → Rolling updates: max unavailable = 0%, max surge = 25% (zero-downtime deploy)
  → Internal ALB routes from web tier to app tier

DATABASE TIER (high availability):
  Aurora PostgreSQL:
  → Multi-AZ: 6 storage copies across 3 AZs (automatic)
  → Writer endpoint: always points to primary
  → Reader endpoint: load balances across up to 15 read replicas
  → Failover: < 30 seconds to a read replica becoming the new primary
  
  ElastiCache Redis (session + cache):
  → Cluster mode enabled: shards distributed across 3 AZs
  → Multi-AZ: each shard has a replica in a different AZ
  → Automatic failover: replica promoted if primary fails

FAILURE SCENARIOS AND RESPONSES:

Scenario: AZ-B fails (one of three AZs goes down)
  ALB: still routes to instances in AZ-A and AZ-C
  ASG: detects AZ-B instances unhealthy → launches replacements in AZ-A and AZ-C
  Aurora: writer was in AZ-B → automatic failover to replica in AZ-A (< 30s)
  Redis: primary shards in AZ-B → failover to replicas in AZ-A and AZ-C
  Result: ~30 second impact, then fully operational at reduced capacity
  Recovery: ASG eventually launches new instances when AZ-B recovers

Scenario: A bad deployment causes 500 errors
  ALB health checks: instances start failing /health → ALB deregisters them
  If ALL instances fail: ALB returns 503 (better than hanging)
  Auto rollback: CodeDeploy detects error rate > 5% → rolls back deployment
  Result: Service degraded for ~5-10 minutes, then rolled back automatically
```

---

### Q4: What is the difference between fault tolerance and high availability?

```
HIGH AVAILABILITY (HA):
  Definition: System minimizes downtime, recovering quickly from failures
  Goal: Uptime percentage (99.9% = 8.7 hrs/year downtime, 99.99% = 52 min/year)
  Method: Redundancy + fast failover
  Recovery time: Seconds to minutes (brief impact)
  Cost: Moderate (2-3x instances, Multi-AZ database)
  
  Example: Aurora Multi-AZ → primary fails → replica promoted in 30 seconds
           Application sees 30 seconds of failed connections, then recovers
           This IS highly available (brief impact, automatic recovery)

FAULT TOLERANCE (FT):
  Definition: System continues operating WITHOUT interruption despite failures
  Goal: Zero impact to users even during a failure event
  Method: Active redundancy — multiple components running simultaneously
  Recovery time: Zero (no recovery needed; another component was already active)
  Cost: High (full duplicate capacity, active-active everywhere)
  
  Example: DynamoDB Global Tables in 3 regions — if one region fails,
           Route53 health check triggers, traffic routes to another region
           Users never see an error (the other region was already serving some traffic)

ANALOGY:
  High Availability = spare tire (you pull over, change tire, continue — brief stop)
  Fault Tolerance = run-flat tires (never stop, even with a puncture)

WHEN TO CHOOSE:
  99.9% uptime SLA (8.7 hr/year): Standard Multi-AZ, Auto Scaling → HA sufficient
  99.99% uptime (52 min/year):    Warm standby cross-region → HA + DR
  99.999% (5 min/year):           Active-active multi-region → Fault Tolerant
  99.9999% (30 sec/year):         Very rare; typically only financial exchanges
```

---

## 3. Disaster Recovery — Tiers & Runbooks {#dr}

---

### Q5: Write me a DR runbook for failing over to a secondary region.

**The Story:**
A runbook is the procedure you follow when everything is on fire and adrenaline is spiking.
A great runbook can be executed by someone who has never done it before.
A bad runbook leaves room for judgment calls at exactly the moment you don't want them.

```
DR RUNBOOK — Region Failover to eu-west-1
Version: 2.1
Last tested: [DATE]
Owner: Platform Engineering
Estimated execution time: 15-30 minutes

PRE-CONDITIONS:
  ✅ This runbook triggers when: us-east-1 is unavailable > 5 minutes
  ✅ Decision authority: On-call lead engineer + VP Engineering approval for prod
  ✅ Communication: Post in #incidents and #war-room before starting

STEP 1 — DECLARE INCIDENT (T+0 to T+2)
  □ Post in #incidents: "Initiating DR failover to eu-west-1. ETA: 15 minutes."
  □ Page secondary on-call if primary is affected
  □ Start incident timeline document

STEP 2 — VERIFY TARGET REGION (T+2 to T+5)
  □ Login to AWS Console → switch to eu-west-1
  □ Check Aurora Global DB secondary cluster status:
    RDS Console → aurora-global-cluster → eu-west-1 cluster → Status = "Available"
    Lag: check "AuroraGlobalDBReplicationLag" metric → should be < 60 seconds
  □ Check EKS cluster in eu-west-1: kubectl get nodes (should return nodes in Ready state)
  □ Check ALB in eu-west-1: status = Active

STEP 3 — PROMOTE AURORA IN eu-west-1 (T+5 to T+8)
  Option A (console):
    RDS → eu-west-1 Aurora cluster → Actions → Remove from Global Cluster
    This promotes it to standalone primary (read+write)
    Wait: ~60 seconds for promotion to complete
  
  Option B (CLI — faster):
    aws rds remove-from-global-cluster \
      --global-cluster-identifier global-prod-cluster \
      --db-cluster-identifier aurora-eu-west-1-cluster \
      --region us-east-1
  
  □ Verify: connect to eu-west-1 cluster endpoint → can INSERT a test row

STEP 4 — UPDATE APPLICATION CONFIGURATION (T+8 to T+12)
  □ Secrets Manager (eu-west-1) already has DB credentials for eu-west-1 cluster
    (These should be pre-configured — do NOT do this for the first time during an incident)
  □ Verify EKS pods in eu-west-1 are running and healthy:
    kubectl get pods -n production | grep -v Running
    (should return nothing)

STEP 5 — ACTIVATE TRAFFIC ROUTING (T+12 to T+15)
  □ Route 53 → hosted zone → primary record
    Current: "A record → us-east-1 ALB" (FAILOVER PRIMARY)
    Status: us-east-1 health check is "Unhealthy" (this triggered DR)
    → Route53 should already be routing to eu-west-1 if health check is failing
    If not automatic: 
    Change: Set primary record TTL to 60s (if not already)
    Verify: dig +short api.mycompany.com → should return eu-west-1 ALB IP
  
  □ CloudFront origin: update to eu-west-1 ALB if applicable

STEP 6 — VALIDATE (T+15 to T+20)
  □ Smoke test: curl https://api.mycompany.com/health → {"status":"ok","region":"eu-west-1"}
  □ Place a test order → verify it appears in Aurora eu-west-1
  □ Check CloudWatch: eu-west-1 ALB 5xx error rate < 0.1%
  □ Check: P99 latency within normal range

STEP 7 — COMMUNICATE (T+20)
  □ Post in #incidents: "Failover complete. Service restored in eu-west-1."
  □ Notify stakeholders (engineering leadership, customer success)
  □ Begin post-mortem scheduling

POST-INCIDENT:
  □ Root cause analysis of us-east-1 failure
  □ Plan failback when us-east-1 recovers
  □ Update runbook with any steps that were unclear
  □ Schedule next DR test (within 30 days)
```

---

## 4. Database Resilience Patterns {#dbresil}

---

### Q6: How do you prevent database connection exhaustion at scale?

**The Story:**
Your app has 100 ECS tasks, each keeping 10 database connections open = 1,000 connections.
Aurora PostgreSQL's default max_connections is 5,000 (for a db.r5.4xlarge). Seems fine.

Black Friday: you scale to 2,000 ECS tasks × 10 connections = 20,000 connections requested.
Aurora rejects connections. App fails. Revenue loss. This is connection exhaustion.

```
SOLUTION: RDS PROXY

What it does:
  → Sits between your application and Aurora
  → Maintains a small, fixed connection pool TO Aurora (e.g., 100 connections)
  → Handles thousands of application connections via connection multiplexing
  → Application thinks it's talking to Aurora; actually talking to RDS Proxy

Architecture:
  2,000 ECS tasks × 10 app connections = 20,000 connections to RDS Proxy
                                                    ↓
                                              RDS PROXY (proxies + multiplexes)
                                                    ↓
                                    100 real connections to Aurora cluster

Additional benefits:
  → Automatic connection retry on failover (app doesn't see failover blip)
  → IAM authentication for all DB connections (no password in app)
  → Reduced failover time: proxy maintains connections during Aurora failover
  → SSL/TLS encryption enforced

Configuration tips:
  → idle_client_timeout: close idle app connections after 1800s
  → connection_borrow_timeout: how long to wait for a connection from pool (default: 120s)
  → max_connections_percent: cap how much of Aurora's max_connections Proxy uses
```

---

### Q7: Explain read/write splitting and how to implement it.

```
PROBLEM:
  Production Aurora cluster is CPU-bound.
  Investigation: 80% of queries are SELECT (read) — reports, dashboards, search.
  Only 20% are writes. The writer instance handles all 100%.

SOLUTION: READ/WRITE SPLITTING

  WRITES → Writer Endpoint:  aurora-cluster.cluster-xxx.us-east-1.rds.amazonaws.com
  READS  → Reader Endpoint:  aurora-cluster.cluster-ro-xxx.us-east-1.rds.amazonaws.com
  
  Aurora automatically load-balances read traffic across all read replicas.

IMPLEMENTATION OPTIONS:

Option A — Application-level (most control):
  # Python SQLAlchemy example
  write_engine = create_engine(WRITER_ENDPOINT_URL)
  read_engine = create_engine(READER_ENDPOINT_URL)
  
  def place_order(order_data):
      with write_engine.connect() as conn:  # always use writer
          conn.execute(INSERT_ORDER, order_data)
  
  def get_order_history(user_id):
      with read_engine.connect() as conn:   # can use reader
          return conn.execute(SELECT_ORDERS, user_id).fetchall()

Option B — ProxySQL / PgBouncer (middleware):
  All connections → ProxySQL
  ProxySQL detects: SELECT? → route to reader endpoint
                   INSERT/UPDATE/DELETE? → route to writer endpoint
  Application doesn't change; proxy handles splitting

Option C — Application framework (Spring, Django ORM):
  @Transactional(readOnly = true)  // Java Spring → uses read datasource
  public List<Order> getOrders() { ... }
  
  @Transactional  // default readOnly=false → uses write datasource
  public void placeOrder() { ... }

CAVEATS:
  → Replication lag: reads from replicas may be slightly behind (< 100ms usually)
  → "Read your own writes" problem: user places order, immediately queries — 
     might not see their own order if query hits replica with 50ms lag
  → Solution: Use Session Consistency feature (Aurora routes to writer for same session after write)
```

---

## 5. Chaos Engineering {#chaos}

---

### Q8: What is chaos engineering and how do you practice it on AWS?

**The Story:**
Netflix coined the term with their "Chaos Monkey" — a tool that randomly terminated production
EC2 instances to force their engineers to build systems resilient enough to survive it.
The insight: if you don't discover your failures in a controlled way, you discover them during
a real incident at the worst possible time.

```
CHAOS ENGINEERING MATURITY MODEL:

Level 1 — KNOWN FAILURES (start here):
  "We think we're resilient to AZ failure. Let's verify."
  → Terminate all instances in one AZ
  → Verify Auto Scaling replaces them, traffic continues
  → Measure: how many errors? How long to recover?

Level 2 — REALISTIC FAILURES:
  "What if the database is slow but not dead?"
  → Inject 2 second latency on all DB connections
  → Does the app time out gracefully? Or do threads pile up?
  → This is harder to handle than a complete failure (systems often do worse with slowness)

Level 3 — COMPLEX SCENARIOS:
  "What if the payment service is slow AND we're getting a traffic spike?"
  → Multiple simultaneous failure injections
  → Tests system behavior under compound failures

AWS FAULT INJECTION SIMULATOR (FIS):

  Create an experiment template:
  Actions:
    aws:ec2:terminate-instances
      → targets: instances tagged Env=Production, AZ=us-east-1a
      → filter: 50% of matching instances
    
    aws:rds:failover-db-cluster
      → targets: aurora-prod-cluster
      → trigger: after 60 seconds
  
  Stop conditions (safety brakes!):
    → Stop if: CloudWatch alarm "ProductionErrorRate" > 5% triggers
    → This prevents the experiment from causing real damage beyond acceptable limits
  
  Duration: 10 minutes
  
  Run on: STAGING (always) before PRODUCTION
  Production chaos: requires VP approval, during low-traffic hours only

GAME DAY PROCESS:
  1. Define hypothesis: "If AZ-B fails, all orders will succeed within 30 seconds"
  2. Define blast radius: "We'll only affect 20% of traffic"
  3. Define rollback: "We can restore in < 5 minutes"
  4. Run experiment
  5. Measure actual vs expected
  6. Fix any gaps found
  7. Re-run until hypothesis is confirmed
  8. Document in runbook: "Validated: AZ failure resilient as of [date]"
```

---

## 6. Load Balancing Deep Dive {#lb}

---

### Q9: Explain the three AWS load balancers and when to use each.

```
APPLICATION LOAD BALANCER (ALB) — Layer 7
  Protocol: HTTP, HTTPS, WebSocket, HTTP/2, gRPC
  
  Routing capabilities:
    Path-based:  /api/* → api-service, /web/* → web-service
    Host-based:  api.company.com → api-cluster, web.company.com → web-cluster
    Header-based: X-User-Type: premium → premium-tier targets
    Query string: ?version=2 → v2-service
  
  Health checks: HTTP/HTTPS GET to a path (e.g., /health)
  
  Authentication:
    → Cognito User Pools: offload OAuth/OIDC to ALB (no code change in app)
    → OIDC: integrate with any identity provider
  
  Use when:
    ✅ Microservices with multiple routes to different backends
    ✅ HTTP/gRPC APIs
    ✅ Need WAF integration (WAF only supports ALB and CloudFront)
    ✅ WebSocket connections (chat, real-time apps)

NETWORK LOAD BALANCER (NLB) — Layer 4
  Protocol: TCP, UDP, TLS
  
  Key differentiators:
    → Ultra-low latency (< 100 microseconds vs ALB's ~1ms)
    → Static IPs / Elastic IPs (ALB IPs change — NLB IPs are fixed)
    → Handles millions of requests per second
    → Preserves source IP (backend sees real client IP, not LB IP)
  
  Use when:
    ✅ Applications that need static IP (customer firewall whitelisting)
    ✅ Non-HTTP: MQTT, AMQP, game server UDP, financial FIX protocol
    ✅ TLS passthrough (NLB doesn't decrypt — backend handles TLS)
    ✅ AWS PrivateLink: must use NLB as the endpoint service
    ✅ Extremely high performance / low latency requirements

GATEWAY LOAD BALANCER (GWLB) — Layer 3
  Protocol: IP (all IP protocols)
  
  What it does:
    → Routes all network traffic through third-party virtual appliances
    → Transparent bump-in-the-wire
  
  Use when:
    ✅ Running Palo Alto, Check Point, Fortinet, or other virtual firewalls on AWS
    ✅ You bought a software-defined security appliance and need to inline it
    ✅ Compliance requires physical-like inspection of all traffic
  
  Architecture:
    VPC traffic → GWLB Endpoint → GWLB → Firewall instances → GWLB → traffic continues

CHOOSING:
  Web application: ALB
  High performance TCP: NLB
  Virtual appliances / inspection: GWLB
  Both on same service: NLB in front of ALB (static IP + application routing)
```

---

## 7. Scripting for Scaling & Automation {#scripting}

---

### Q10: Show me how to write a Lambda function that automatically scales ECS based on a custom metric.

```python
# Lambda function: custom-ecs-scaler
# Trigger: CloudWatch Events (every 1 minute)
# Purpose: Scale ECS service based on SQS queue depth

import boto3
import os

ecs = boto3.client('ecs')
sqs = boto3.client('sqs')
cloudwatch = boto3.client('cloudwatch')

CLUSTER_NAME = os.environ['ECS_CLUSTER']
SERVICE_NAME = os.environ['ECS_SERVICE']
QUEUE_URL = os.environ['SQS_QUEUE_URL']
MESSAGES_PER_TASK = int(os.environ.get('MESSAGES_PER_TASK', '100'))
MIN_TASKS = int(os.environ.get('MIN_TASKS', '1'))
MAX_TASKS = int(os.environ.get('MAX_TASKS', '50'))

def handler(event, context):
    # Get current queue depth
    queue_attrs = sqs.get_queue_attributes(
        QueueUrl=QUEUE_URL,
        AttributeNames=['ApproximateNumberOfMessages']
    )
    queue_depth = int(queue_attrs['Attributes']['ApproximateNumberOfMessages'])
    
    # Calculate desired task count
    desired = max(MIN_TASKS, min(MAX_TASKS, queue_depth // MESSAGES_PER_TASK + 1))
    
    # Get current task count
    service = ecs.describe_services(
        cluster=CLUSTER_NAME,
        services=[SERVICE_NAME]
    )['services'][0]
    current = service['desiredCount']
    
    # Only update if different (avoid unnecessary API calls)
    if desired != current:
        ecs.update_service(
            cluster=CLUSTER_NAME,
            service=SERVICE_NAME,
            desiredCount=desired
        )
        print(f"Scaled {SERVICE_NAME}: {current} → {desired} tasks (queue depth: {queue_depth})")
    
    # Publish custom metric for visibility
    cloudwatch.put_metric_data(
        Namespace='CustomScaling',
        MetricData=[{
            'MetricName': 'DesiredTaskCount',
            'Value': desired,
            'Unit': 'Count',
            'Dimensions': [{'Name': 'Service', 'Value': SERVICE_NAME}]
        }]
    )
    
    return {'desired': desired, 'current': current, 'queue_depth': queue_depth}
```

---

### Q11: Write a Python script to check if all EC2 instances in an ASG are healthy and restart any that aren't.

```python
#!/usr/bin/env python3
# asg-health-check.py — Run from CloudWatch Events or manually

import boto3
import argparse
import json
from datetime import datetime

autoscaling = boto3.client('autoscaling')
ec2 = boto3.client('ec2')
sns = boto3.client('sns')

def check_asg_health(asg_name, alert_topic_arn=None):
    # Get all instances in the ASG
    response = autoscaling.describe_auto_scaling_groups(
        AutoScalingGroupNames=[asg_name]
    )
    
    if not response['AutoScalingGroups']:
        raise ValueError(f"ASG not found: {asg_name}")
    
    asg = response['AutoScalingGroups'][0]
    instances = asg['Instances']
    
    unhealthy = []
    for instance in instances:
        if instance['HealthStatus'] != 'Healthy':
            unhealthy.append({
                'instance_id': instance['InstanceId'],
                'health_status': instance['HealthStatus'],
                'lifecycle_state': instance['LifecycleState']
            })
    
    report = {
        'asg_name': asg_name,
        'total_instances': len(instances),
        'unhealthy_count': len(unhealthy),
        'unhealthy_instances': unhealthy,
        'timestamp': datetime.utcnow().isoformat()
    }
    
    if unhealthy:
        print(f"⚠️  Found {len(unhealthy)} unhealthy instances in {asg_name}")
        for inst in unhealthy:
            print(f"  - {inst['instance_id']}: {inst['health_status']} ({inst['lifecycle_state']})")
        
        # Set health to Unhealthy to trigger ASG replacement
        for inst in unhealthy:
            if inst['lifecycle_state'] == 'InService':
                autoscaling.set_instance_health(
                    InstanceId=inst['instance_id'],
                    HealthStatus='Unhealthy',
                    ShouldRespectGracePeriod=False
                )
                print(f"  → Marked {inst['instance_id']} Unhealthy — ASG will replace it")
        
        # Send alert
        if alert_topic_arn:
            sns.publish(
                TopicArn=alert_topic_arn,
                Subject=f"ASG Health Alert: {asg_name}",
                Message=json.dumps(report, indent=2)
            )
    else:
        print(f"✅ All {len(instances)} instances healthy in {asg_name}")
    
    return report

if __name__ == '__main__':
    parser = argparse.ArgumentParser()
    parser.add_argument('--asg', required=True, help='ASG name')
    parser.add_argument('--topic', help='SNS topic ARN for alerts')
    args = parser.parse_args()
    
    check_asg_health(args.asg, args.topic)
```

---

## Resilience Quick Reference

```
THE FOUR GOLDEN SIGNALS (SRE fundamentals):
  Latency:    How long does a request take? (measure P50, P95, P99 — NOT average)
  Traffic:    How much demand is the system receiving? (requests/sec)
  Errors:     What fraction of requests are failing? (HTTP 5xx rate)
  Saturation: How "full" is the system? (CPU %, queue depth, memory %)
  
  Set alarms on ALL FOUR. If any deviates significantly → investigate.

SLO / SLI / SLA DEFINITIONS:
  SLI (Service Level Indicator): a measurement
     "Success rate = successful_requests / total_requests"
  SLO (Service Level Objective): your internal target
     "We aim for 99.9% success rate over any 28-day window"
  SLA (Service Level Agreement): contractual commitment
     "We guarantee 99.5% uptime or we issue credits"
  
  Note: SLA < SLO < actual capability (buffer for safety)

CHAOS INJECTION TARGETS (in priority order):
  1. AZ failure (most common, best known AWS failure mode)
  2. Single instance failure (happens constantly — should be invisible)
  3. Database failover (RDS/Aurora failover)
  4. Dependency timeout (downstream service slow, not dead)
  5. Region failure (rare, catastrophic — test annually in staging copy)

HA DESIGN CHECKLIST:
  ✅ No single instance is serving production traffic alone
  ✅ Every component is deployed in at least 2 AZs
  ✅ Auto Scaling Group with health checks replaces failed instances
  ✅ Load balancer health checks match application readiness (not just OS health)
  ✅ Database has Multi-AZ or Global replication
  ✅ Circuit breakers on all external dependencies
  ✅ Retry with exponential backoff on all API calls
  ✅ Timeouts defined on every network call
  ✅ DLQ on every SQS queue
  ✅ Runbooks exist AND have been tested in the last 90 days
```

---

## Final Interview Checklist — Senior/Principal Level

```
SYSTEM DESIGN FRAMEWORK (use this structure for every design question):
  1. Clarify: users, scale, latency SLA, geography, compliance
  2. High-level: draw the components without implementation detail
  3. Data layer: what DB, how does it scale, what's the schema
  4. Scale: where are the bottlenecks? How do you handle 10x traffic?
  5. Failure modes: what breaks first? How does the system behave?
  6. Security: data at rest, in transit, authentication, authorization
  7. Operations: how do you deploy, monitor, and debug this?
  8. Cost: rough estimate of monthly cost at stated scale

PHRASES THAT IMPRESS:
  "I'd start simple and add complexity only where justified by the requirement..."
  "The trade-off here is between X and Y — given the stated constraint, I'd choose..."
  "I'd want to validate this with load testing before committing to this design..."
  "One thing I'd add to the monitoring is the four golden signals..."
  "For compliance, I'd make sure we have CloudTrail + Config recording in every account..."
  "I've seen this pattern fail when... so I'd mitigate that by..."
```

---

*End of InterviewQnA Series — Good luck! You've got this. 🚀*

*Review order: Beginner → Intermediate → Advanced → DevOps → Networking → Landing Zones → Migration → Resilience*
