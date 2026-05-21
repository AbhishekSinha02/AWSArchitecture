# AWS Interview Q&A — Networking, Hub-Spoke & Hybrid Connectivity

> **The Network Story:** Everything in AWS runs on a network. The compute, the databases,
> the messaging — it all flows through VPCs, subnets, route tables, and gateways. A flawed
> network design becomes a ceiling that limits everything above it. Get networking right and
> you get security, performance, and scalability as a side effect.

---

## Table of Contents
1. [VPC Deep Dive](#vpc)
2. [Transit Gateway — Hub-Spoke Architecture](#tgw)
3. [Hybrid Connectivity — Direct Connect & VPN](#hybrid)
4. [DNS at Enterprise Scale](#dns)
5. [Network Security](#netsec)
6. [Advanced Routing Patterns](#routing)

---

## 1. VPC Deep Dive {#vpc}

---

### Q1: Walk me through designing a VPC for a three-tier production application.

**The Story:**
You're asked to design the network for a new microservices platform. Here's how a senior
architect thinks through it, layer by layer.

```
DESIGN DECISIONS (in order):

Step 1 — CIDR Selection
  Use /16 for VPC (e.g., 10.10.0.0/16 = 65,536 IPs)
  Reserve /16 ranges per environment, per region — document in IPAM
  DO NOT use 10.0.0.0/8 ranges already used on-prem (they'll conflict with VPN/DX)

  Subnet design:
    Public:   10.10.1.0/24, 10.10.2.0/24, 10.10.3.0/24  (one per AZ)
    Private:  10.10.11.0/24, 10.10.12.0/24, 10.10.13.0/24
    Isolated: 10.10.21.0/24, 10.10.22.0/24, 10.10.23.0/24 (DB tier)
    Spare:    /24 blocks reserved for future use

Step 2 — GATEWAYS
  Internet Gateway (IGW): attached to VPC (one per VPC)
  NAT Gateways: one per AZ in PUBLIC subnets
    → Private subnet route: 0.0.0.0/0 → NAT GW in SAME AZ
    → Each AZ has its own NAT GW (otherwise AZ failure = lost internet egress for app tier)
  VPC Endpoints (Gateway type, free):
    → S3 endpoint: route table entry → no NAT GW charge for S3
    → DynamoDB endpoint: same benefit

Step 3 — ROUTE TABLES
  Public subnet RT:   0.0.0.0/0 → IGW
  Private subnet RT:  0.0.0.0/0 → NAT GW (same AZ), 10.10.0.0/16 → local
  Isolated subnet RT: 10.10.0.0/16 → local ONLY (no internet route)

Step 4 — SECURITY GROUPS (defense in depth)
  ALB SG:         inbound 443 from 0.0.0.0/0 (internet)
  App SG:         inbound 8080 from ALB SG only (no direct internet)
  DB SG:          inbound 5432 from App SG only (not from ALB, not from internet)
  Bastion/SSM SG: no open inbound ports (use SSM Session Manager instead)

Step 5 — VPC FLOW LOGS
  Enable on VPC → CloudWatch Logs or S3
  Query with Athena: "show me all traffic to port 3306 that was REJECTED"
```

---

### Q2: What is a VPC Endpoint and why should you use them?

**The Story:**
Without VPC endpoints, your private EC2 instance calling `s3.amazonaws.com` sends traffic like this:
Private subnet → NAT Gateway → Internet Gateway → Public Internet → S3.

You're paying for NAT Gateway data processing ($0.045/GB), and your S3 traffic is leaving your
VPC and travelling over the public internet — even though S3 and EC2 are both "in AWS."

VPC Endpoints change this:
```
WITHOUT ENDPOINT:  EC2 → NAT GW → IGW → Internet → S3   ($$$, not private)
WITH ENDPOINT:     EC2 → VPC Endpoint → S3               (free, stays in AWS backbone)

TWO TYPES:

Gateway Endpoints (free, only S3 and DynamoDB):
  → Entry in route table: "traffic to S3 → go to endpoint, not NAT GW"
  → Best practice: ALWAYS create these for any VPC — zero cost, immediate savings

Interface Endpoints (costs ~$7/month per endpoint per AZ):
  → Creates an ENI (private IP) in your subnet
  → DNS resolves the service name to a private IP
  → Supported for 100+ services: Secrets Manager, KMS, SSM, ECR, CloudWatch...
  → Required for: services accessed by private Lambda functions,
                  air-gapped environments (no internet at all)
  → Enable private DNS: internal DNS for secretsmanager.us-east-1.amazonaws.com
    resolves to the private endpoint IP automatically
```

**Cost Example for Interview:**
"Our Lambda functions in a private VPC were downloading container images from ECR every cold start.
At 10 million invocations/month × 50 MB images = 500 TB of NAT Gateway data transfer = $22,500/month.
Adding an ECR Interface Endpoint cost $14/month and eliminated the NAT GW charge entirely."

---

## 2. Transit Gateway — Hub-Spoke Architecture {#tgw}

---

### Q3: Explain Transit Gateway and why it replaced VPC Peering at scale.

**The Story — The Problem with VPC Peering:**

Imagine you're growing from 5 VPCs to 50 VPCs. With VPC peering, every VPC that needs to talk
to every other VPC needs its own peering connection. That's N*(N-1)/2 peering connections.

```
5 VPCs:  10 peering connections (manageable)
20 VPCs: 190 peering connections (painful)
50 VPCs: 1,225 peering connections (impossible to manage)

Also: VPC Peering is NOT transitive.
  VPC-A ←→ VPC-B ←→ VPC-C
  VPC-A CANNOT talk to VPC-C through B. Need direct peering A-C.
```

**Transit Gateway — The Hub:**

```
TRANSIT GATEWAY HUB-SPOKE ARCHITECTURE:

        ┌──────────────────────────────────────────┐
        │         TRANSIT GATEWAY (Hub)            │
        │    TGW Route Table:                      │
        │    10.10.0.0/16 → VPC-Prod attachment    │
        │    10.20.0.0/16 → VPC-Dev attachment     │
        │    10.30.0.0/16 → VPC-Shared attachment  │
        │    0.0.0.0/0    → VPN/DX attachment      │
        └──────────┬───────────────────────────────┘
                   │ attachments
       ┌───────────┼──────────────────┐
       ▼           ▼                  ▼
   VPC-Prod    VPC-Dev           VPC-Shared        VPN (on-prem)
  (10.10/16)  (10.20/16)        (10.30/16)        Direct Connect

Key properties:
  ✅ Single attachment per VPC → TGW routes to all others
  ✅ Transitive routing (unlike VPC Peering)
  ✅ Up to 5,000 VPC attachments per TGW
  ✅ Cross-account: share via AWS RAM (Resource Access Manager)
  ✅ Cross-region: TGW Peering for global networks
  ❌ Cost: $0.05/GB processed + $0.05/attachment/hour
     → At scale, inter-VPC transfer costs can be significant
```

---

### Q4: Design a Hub-Spoke network for an enterprise with 50 VPCs across dev, staging, and prod.

**The Story:**
This is the most common enterprise AWS networking question at senior/principal level.
The key insight: don't connect everything to everything. Segment by environment and security domain.

```
ENTERPRISE HUB-SPOKE DESIGN:

ACCOUNT STRUCTURE (each box is a separate AWS account):
┌─────────────────────────────────────────────────────────────────────┐
│                        NETWORK HUB ACCOUNT                          │
│                                                                     │
│  Transit Gateway (us-east-1)                                        │
│  │                                                                  │
│  ├── TGW Route Table: PRODUCTION                                    │
│  │     → only prod VPCs talk to each other + shared services       │
│  │     → CANNOT reach dev or staging                               │
│  │                                                                  │
│  ├── TGW Route Table: NON-PRODUCTION                               │
│  │     → dev and staging VPCs                                      │
│  │     → isolated from prod                                        │
│  │                                                                  │
│  ├── TGW Route Table: SHARED SERVICES                              │
│  │     → DNS resolver (Route 53 Resolver endpoints)               │
│  │     → Active Directory                                          │
│  │     → Monitoring/logging (Splunk, Grafana)                      │
│  │     → Accessible from BOTH prod and non-prod                    │
│  │                                                                  │
│  └── TGW Route Table: ON-PREMISES                                  │
│        → VPN/Direct Connect attachments                            │
│        → on-prem can reach shared services + prod only             │
│                                                                     │
│  Egress VPC (centralized):                                         │
│    All outbound internet traffic goes through one VPC              │
│    with Network Firewall → full inspection before leaving AWS      │
└─────────────────────────────────────────────────────────────────────┘
         │                    │                    │
┌────────▼─────┐  ┌──────────▼────┐  ┌────────────▼──────┐
│   PROD       │  │ NON-PROD      │  │  SHARED SERVICES  │
│  ACCOUNT(S)  │  │ ACCOUNT(S)    │  │  ACCOUNT           │
│  VPC-App     │  │ VPC-Dev       │  │  VPC-DNS           │
│  VPC-Data    │  │ VPC-Staging   │  │  VPC-AD            │
│  VPC-ML      │  │ VPC-QA        │  │  VPC-Monitoring    │
└──────────────┘  └───────────────┘  └───────────────────┘

ROUTING ISOLATION EXAMPLE:
  Dev VPC can reach: Shared Services VPC
  Dev VPC CANNOT reach: Prod App VPC (different route table, no route)
  → Even if a dev somehow gets prod DB credentials, network blocks the connection
```

---

## 3. Hybrid Connectivity {#hybrid}

---

### Q5: Direct Connect vs VPN — how do you choose?

**The Story:**
You need to connect your on-premises data center to AWS. You have two options.
Each has a different risk profile, lead time, cost, and use case.

```
SITE-TO-SITE VPN:
  What it is: Encrypted tunnel over the public internet (IPSec)
  Setup time: 15 minutes (just configure VGW and download customer gateway config)
  Bandwidth: 1.25 Gbps per tunnel (can use ECMP with multiple tunnels: up to 10 Gbps)
  Reliability: Subject to internet conditions (jitter, latency variance, packet loss)
  Cost: $0.05/connection/hour + data transfer
  Use for:
    ✅ Quick connectivity (proof of concept, disaster recovery backup)
    ✅ Remote branch offices
    ✅ Backup path alongside Direct Connect
    ✅ Small data transfer volumes

DIRECT CONNECT:
  What it is: Dedicated private fiber connection from your data center to AWS
  Setup time: 30-90 days (physical fiber provisioning through DX partner)
  Bandwidth: 1, 10, or 100 Gbps (dedicated); 50 Mbps-10 Gbps hosted connections
  Reliability: Consistent (not over internet), < 1ms latency variance
  Cost: Port-hour charges + data transfer (but lower $/GB than internet egress)
  Use for:
    ✅ High-volume data transfer (database migrations, backups, analytics)
    ✅ Low-latency, consistent performance requirements
    ✅ Hybrid workloads with sensitive data (financial, healthcare)
    ✅ Long-term production hybrid connectivity
    ❌ NOT encrypted by default — add MACsec or run VPN over DX for encryption

BEST PRACTICE — ENTERPRISE:
  Primary path:  Direct Connect (1 or 2 connections via different DX partners = HA)
  Backup path:   Site-to-site VPN (auto-failover via BGP)
  Architecture:  DX + VPN in Active/Passive BGP configuration
```

---

### Q6: What is Direct Connect Gateway and when do you need it?

**The Story:**
You have Direct Connect set up, connected to a Virtual Private Gateway (VGW) in us-east-1.
Now business grows — you need to connect the same on-premises DC to VPCs in eu-west-1.
Without DX Gateway: you'd need a second Direct Connect connection to the European colocation.

DX Gateway is a global construct that lets ONE Direct Connect connection reach VPCs in MULTIPLE regions:

```
ON-PREMISES DC
    │
    │ Direct Connect (physical fiber)
    │
DX LOCATION (e.g., Equinix Chicago)
    │
DIRECT CONNECT GATEWAY (global AWS object — not in any region)
    │
    ├──── Transit Gateway (us-east-1) ──── VPCs in us-east-1
    ├──── Transit Gateway (eu-west-1) ──── VPCs in eu-west-1
    └──── VGW (ap-southeast-1) ────────── VPC in Singapore

Result:
  One physical DX connection → access to resources in 3 regions
  VPCs in all regions can privately access on-prem resources
  Data transfer charges still apply per region
```

---

## 4. DNS at Enterprise Scale {#dns}

---

### Q7: How does DNS resolution work across a complex multi-VPC, hybrid AWS environment?

**The Story:**
DNS is the phone book of your network. In a hybrid enterprise, you have:
- AWS services with Route 53 Private Hosted Zones
- On-premises DNS servers (often Active Directory DNS)
- Multiple VPCs that need to resolve both

Without Route 53 Resolver: it's a mess. Each VPC has its own resolver (169.254.169.253), and
on-premises DNS can't query it. On-premises machines can't resolve `db.internal.mycompany.com`
if that's hosted in Route 53.

**Route 53 Resolver — The Bridge:**

```
COMPONENTS:
  Inbound Endpoint:  Creates ENIs in your VPC
                     → On-premises DNS servers forward queries to these IPs
                     → Resolves Route 53 Private Hosted Zone records for on-prem

  Outbound Endpoint: Creates ENIs in your VPC
                     → Route 53 Resolver forwards specific domains to on-prem DNS
                     → Resolves on-prem hostnames (e.g., app.corp.internal) from AWS

  Resolver Rules:    Define forwarding rules
                     → "Queries for *.corp.internal → forward to 10.0.1.10 (on-prem AD DNS)"
                     → Share rules with all accounts via AWS RAM

FULL PICTURE:

  On-Premises DNS Server (10.0.1.10)
    Receives query: "what is the IP for api.prod.aws.mycompany.com?"
    Has conditional forwarder: *.aws.mycompany.com → Route 53 Inbound Endpoint (10.10.1.100)
    Forwards to Route 53 Inbound Endpoint
    → Route 53 resolves: Private Hosted Zone "aws.mycompany.com" → returns 10.10.2.50
    On-prem machine gets the answer, connects to AWS resource

  AWS Lambda Function (in VPC)
    Needs to reach: "app.corp.internal" (on-prem Active Directory)
    Route 53 Resolver: has outbound rule: *.corp.internal → 10.0.1.10 (on-prem DNS)
    Forwards to on-prem DNS → gets answer → Lambda connects to on-prem resource

SHARING ACROSS ACCOUNTS:
  → Create Resolver rules in Network Hub account
  → Share via AWS RAM to all spoke accounts
  → All VPCs automatically use the same forwarding rules
  → Centrally managed, changes propagate everywhere
```

---

## 5. Network Security {#netsec}

---

### Q8: What is AWS Network Firewall and how does it differ from Security Groups?

```
SECURITY GROUPS vs NACLs vs NETWORK FIREWALL:

SECURITY GROUPS:
  Scope: Instance/ENI level
  Capability: Allow rules only, stateful, up to 60 inbound + 60 outbound rules
  Use for: Basic instance-level filtering (port 443 to specific SG, etc.)
  Cannot: Deep packet inspection, stateful protocol parsing, IPS/IDS

NACLs:
  Scope: Subnet level
  Capability: Allow and Deny, stateless, ordered rules
  Use for: Subnet-level blocking (block specific IP ranges)
  Cannot: Application-layer inspection, protocol detection

NETWORK FIREWALL:
  Scope: VPC level (deployed in a dedicated subnet per AZ)
  Capability: 
    → Stateful packet inspection (tracks TCP sessions)
    → Application-layer filtering (HTTP, TLS SNI, DNS inspection)
    → Intrusion Prevention System (Suricata-compatible rules)
    → Domain-based filtering: "allow traffic to *.amazonaws.com only"
    → TLS inspection (decrypt, inspect, re-encrypt)
  Use for:
    → Egress filtering (prevent data exfiltration)
    → Centralized inspection in Hub VPC
    → Compliance: PCI DSS requires stateful firewall logging

DEPLOYMENT PATTERN (centralized egress):

  App VPC (private subnet)
    │
    │ TGW route: 0.0.0.0/0 → Egress VPC attachment
    ▼
  TGW
    │
    ▼
  EGRESS VPC
    ├── Network Firewall subnet (inspects traffic)
    │     Allows: *.amazonaws.com, known patch servers
    │     Blocks: everything else
    └── NAT Gateway (outbound to internet)
  
  All 50 VPCs share one centralized egress point = one policy to manage
```

---

## 6. Advanced Routing Patterns {#routing}

---

### Q9: Explain BGP in the context of AWS Direct Connect and Transit Gateway.

**The Story:**
BGP (Border Gateway Protocol) is the routing protocol of the internet — and also the language
your Direct Connect speaks to exchange routes with AWS.

```
BGP ON DIRECT CONNECT:

You configure two BGP sessions on your Direct Connect Virtual Interface:
  1. Your customer gateway (your router) advertises on-prem routes to AWS
     "10.0.0.0/8 is reachable from my side"
  2. AWS advertises VPC CIDR ranges back to you
     "10.10.0.0/16 is your VPC in us-east-1"

Your router's routing table: 10.10.0.0/16 via AWS DX interface → sends traffic to VPC
AWS route table: 10.0.0.0/8 via VGW/TGW → sends on-prem traffic through DX

ACTIVE/PASSIVE WITH BGP:
  You have: Direct Connect (primary) + VPN (backup)
  Both advertise the same on-prem routes

  Make DX preferred:
    On the VPN BGP session, prepend your AS-PATH: "65000 65000 65000 10.0.0.0/8"
    More AS hops = less preferred route
    AWS prefers the DX path (shorter AS-PATH)
    
  DX fails → BGP withdraws DX routes → traffic automatically shifts to VPN

ECMP (Equal Cost Multi-Path):
  Two DX connections, same route advertised on both
  AWS splits traffic equally across both connections
  1 Gbps + 1 Gbps = 2 Gbps effective bandwidth
  One fails → other carries all traffic (graceful degradation)
```

---

### Q10: What is PrivateLink and when do you use it instead of VPC Peering?

**The Story:**
You run a SaaS company. Your enterprise customers want to access your API from their private VPCs
without traffic leaving their network. VPC Peering would expose your entire VPC CIDR to theirs —
including your databases, internal services, and other customers' data.

PrivateLink is purpose-built for this:

```
WITHOUT PRIVATELINK:
  Customer VPC ←── VPC Peering ──→ Your Service VPC
  Problem: Customer can route to ANY resource in your VPC
           They can see (or try to reach) your DB, other customer data, etc.

WITH PRIVATELINK:
  Your side: 
    NLB sits in front of your service
    Create a VPC Endpoint Service (attached to NLB)
    Approve which customer accounts can connect

  Customer side:
    Create an Interface VPC Endpoint pointing to your endpoint service
    AWS creates an ENI in their subnet with a private IP
    DNS resolves your service name to that private ENI IP

  Traffic flow:
    Customer app → private ENI (in customer VPC) → AWS PrivateLink fabric → NLB → your service
    
  What the customer CANNOT do:
    → Reach any other resource in your VPC (they only have the endpoint to NLB)
    → Route around the NLB to your DB
    → See your internal network structure

COMMON USE CASES:
  ✅ SaaS providers: offer private access to enterprise customers
  ✅ Internal microservices: expose a service to other accounts without full VPC access
  ✅ Compliance: keep financial data traffic entirely off internet
  ✅ AWS services: 100+ AWS services use PrivateLink for Interface Endpoints
```

---

## Networking Quick Reference

```
CIDR CHEAT SHEET:
  /16 = 65,536 IPs  → VPC size
  /24 = 256 IPs     → Standard subnet
  /28 = 16 IPs      → Lambda ENI subnets (use /28 for endpoint subnets)
  /32 = 1 IP        → Single host route

  AWS RESERVES 5 IPs per subnet:
  .0 = Network, .1 = Router, .2 = DNS, .3 = Reserved, .255 = Broadcast
  /24 subnet = 251 usable IPs (not 256)

BANDWIDTH LIMITS TO KNOW:
  NAT Gateway:          45 Gbps per AZ
  VPC Peering:          No limit (limited by network card)
  TGW:                  50 Gbps per VPC attachment
  Direct Connect:       1, 10, or 100 Gbps
  VPN tunnel:           1.25 Gbps (ECMP: up to 10 tunnels = 12.5 Gbps)
  Interface Endpoint:   10 Gbps (scales with concurrent connections)

ROUTE PRIORITY ORDER (most specific wins):
  /32 > /24 > /16 > /8 > 0.0.0.0/0
  Propagated routes < Static routes (in same route table)
  Local VPC routes ALWAYS win (cannot be overridden)

TROUBLESHOOTING CHECKLIST:
  1. Security Group: correct port/protocol from correct source SG?
  2. NACL: both inbound AND outbound rules? Ephemeral ports (1024-65535)?
  3. Route Table: is there a route to the destination?
  4. Internet Gateway: is IGW attached to VPC?
  5. Public IP: does the instance have a public IP (or EIP)?
  6. DNS: enableDnsHostnames AND enableDnsSupport both true on VPC?
```

---

*Next: [06_LandingZones_MultiAccount.md](./06_LandingZones_MultiAccount.md) — Enterprise governance, Landing Zones, and Control Tower*
