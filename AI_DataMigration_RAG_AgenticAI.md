# AI Project — On-Prem to AWS: Data Migration → RAG + Agentic AI + Governance
> **Interview Target:**  Senior Architect / AI Platform Engineer  
> **Pitch Duration:** 15 minutes  
> **Last Updated:** 2026-05-20

---

## 🗺️ MASTER MIND MAP

```
ON-PREM DATA (Structured + Unstructured)
        │
        ▼
┌─────────────────────────────────────────────────┐
│           PHASE 1: MIGRATION LAYER               │
│  DataSync │ DMS │ Snowball Edge │ Direct Connect │
└─────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────┐
│           PHASE 2: DATA LAKE (S3)                │
│   Bronze (Raw) → Silver (Curated) → Gold (AI)   │
│   Glue Catalog │ Lake Formation │ Macie          │
└─────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────┐
│           PHASE 3: AI INGESTION PIPELINE         │
│  Textract │ Comprehend │ Bedrock Embeddings      │
│  OpenSearch Serverless (Vector Store)            │
└─────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────┐
│      PHASE 4: RAG + AGENTIC AI RUNTIME           │
│  Bedrock Knowledge Bases │ Bedrock Agents        │
│  Claude / Titan │ Step Functions │ Lambda        │
└─────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────┐
│           PHASE 5: AI GOVERNANCE                 │
│  Bedrock Guardrails │ SageMaker Clarify          │
│  Model Monitor │ CloudTrail │ Config │ Macie     │
└─────────────────────────────────────────────────┘
```

---

## 🔤 GLOSSARY — Full Forms for Every Abbreviation in This Document

> **Why this exists:** Tech docs assume you know every acronym. You should not have to guess.
> If you are reading this to learn or prepare for an interview, every term here is explained
> in plain English so you can also explain it to a non-technical stakeholder.

### AWS Service Abbreviations
| Short Form | Full Form | Plain English Explanation |
|---|---|---|
| S3 | Simple Storage Service | AWS's object storage — like a giant hard drive in the cloud; stores files, backups, datasets |
| EC2 | Elastic Compute Cloud | Virtual machines (servers) you rent by the hour on AWS |
| ECS | Elastic Container Service | Runs Docker containers — AWS manages the underlying servers for you |
| EKS | Elastic Kubernetes Service | Managed Kubernetes — industry standard for running many containers at scale |
| EMR | Elastic MapReduce | Managed big data cluster (Spark, Hadoop) — for processing terabytes of data in parallel |
| RDS | Relational Database Service | Managed relational databases — AWS handles backups, patching, failover |
| DMS | Database Migration Service | AWS tool to move databases from on-premises or other clouds into AWS |
| SCT | Schema Conversion Tool | Converts database structure (tables, stored procedures) between different DB engines |
| CDC | Change Data Capture | Tracks every INSERT, UPDATE, DELETE on a database in real-time — like a live change log |
| SQS | Simple Queue Service | Message queue — services put messages in, other services read them out; decouples systems |
| SNS | Simple Notification Service | Fan-out messaging — one message can go to many subscribers (email, Lambda, SQS, etc.) |
| KMS | Key Management Service | AWS service that creates and manages encryption keys for protecting your data |
| CMK | Customer Master Key | An encryption key YOU create and fully control in KMS (versus AWS managing it) |
| IAM | Identity and Access Management | Controls who (users, services, apps) can do what in your AWS account |
| VPC | Virtual Private Cloud | Your own isolated private network inside AWS — like having your own data center network |
| ALB | Application Load Balancer | Distributes incoming web traffic across servers; understands HTTP/HTTPS (Layer 7) |
| NLB | Network Load Balancer | Distributes TCP/UDP traffic; ultra-low latency; does not understand HTTP (Layer 4) |
| WAF | Web Application Firewall | Sits in front of your app and blocks malicious web requests (SQL injection, XSS, etc.) |
| API GW | API Gateway | AWS-managed front door for your APIs — handles authentication, throttling, routing |
| CDN | Content Delivery Network | Network of servers worldwide that cache your content close to users (e.g., CloudFront) |
| ACU | Aurora Capacity Unit | Unit of compute power for Aurora Serverless v2 — scales up and down automatically |
| OCU | OpenSearch Capacity Unit | Unit of compute for OpenSearch Serverless — you pay per OCU-hour |
| HSM | Hardware Security Module | A physical device that stores cryptographic keys in tamper-proof hardware |
| FIPS | Federal Information Processing Standards | US government security standards — FIPS 140-2 Level 3 = highest cryptographic certification |
| NFS | Network File System | Linux/Unix protocol for sharing files over a network (e.g., your on-premises NAS storage) |
| SMB | Server Message Block | Windows protocol for sharing files over a network (e.g., Windows file shares) |
| IOPS | Input/Output Operations Per Second | Measures storage speed — how many read/write operations per second a disk can handle |
| TTL | Time To Live | How long a record is kept before being automatically deleted (used in DynamoDB, DNS, caches) |
| DLQ | Dead Letter Queue | A queue that holds messages that failed processing — used for debugging and manual retry |
| FIFO | First In, First Out | Queue ordering guarantee — messages processed in the exact order they were received |
| MWAA | Managed Workflows for Apache Airflow | AWS's managed version of Apache Airflow — no server management needed |
| MSK | Managed Streaming for Apache Kafka | AWS's managed Kafka service — Kafka without managing the cluster yourself |

### AI / ML (Machine Learning) Abbreviations
| Short Form | Full Form | Plain English Explanation |
|---|---|---|
| LLM | Large Language Model | An AI model trained on massive text data that can understand and generate human language |
| RAG | Retrieval-Augmented Generation | Technique where LLM answers are grounded by first fetching relevant documents from a knowledge base |
| OSS | Open Source Software | Software whose source code is publicly available — free to use, modify, and distribute |
| OCR | Optical Character Recognition | Technology that converts scanned images or PDFs into machine-readable text |
| NLP | Natural Language Processing | AI techniques for understanding, interpreting, and generating human language |
| PII | Personally Identifiable Information | Data that can identify a specific person — name, email, phone number, SSN, DOB |
| PHI | Protected Health Information | Medical or health-related data protected by law (HIPAA in the US, PIPEDA in Canada) |
| k-NN | k-Nearest Neighbors | Algorithm to find the K most similar items in a dataset — core of vector search |
| BM25 | Best Match 25 | A keyword-based text ranking algorithm; smarter than simple keyword search |
| RRF | Reciprocal Rank Fusion | Algorithm that combines results from multiple search methods into one better-ranked list |
| SHAP | SHapley Additive exPlanations | Mathematical technique to explain WHICH input features drove a machine learning model's prediction |
| ROUGE | Recall-Oriented Understudy for Gisting Evaluation | Metric to measure how good a text summary is by comparing it to a reference answer |
| RAGAS | RAG Assessment | Open-source Python framework for automatically evaluating how well a RAG pipeline performs |
| ReAct | Reasoning and Acting | An agent pattern where the LLM: Reasons about a problem → Takes an action → Observes result → Repeats |
| A2I | Augmented AI | Amazon service for adding a human review step into ML (Machine Learning) workflows |
| MLOps | Machine Learning Operations | Applying DevOps (CI/CD, monitoring, versioning) practices to machine learning models |
| POC | Proof of Concept | A quick small demo to prove an idea is technically feasible before committing full resources |
| dbt | data build tool | Open-source tool for transforming data in your data warehouse using SQL SELECT statements |
| GPU | Graphics Processing Unit | Hardware originally for gaming graphics, now essential for training and running large AI models |
| HF | Hugging Face | A platform hosting 300,000+ open-source AI models and datasets — "GitHub for AI" |
| LoRA | Low-Rank Adaptation | A technique to fine-tune large language models cheaply by only training a small set of parameters |
| SaaS | Software as a Service | Software delivered over the internet — you pay a subscription, provider manages everything |

### Architecture and Compliance Abbreviations
| Short Form | Full Form | Plain English Explanation |
|---|---|---|
| RPO | Recovery Point Objective | Maximum acceptable data loss — "if disaster strikes, how old can our most recent backup be?" |
| RTO | Recovery Time Objective | Maximum acceptable downtime — "how fast must the system be restored after a failure?" |
| TLS | Transport Layer Security | The encryption protocol used for data in transit — what makes HTTPS secure |
| SLA | Service Level Agreement | A contract defining guaranteed uptime, performance targets, and support response times |
| DAG | Directed Acyclic Graph | A workflow where steps flow forward (directed) and never loop back (acyclic) |
| HPC | High Performance Computing | Workloads that need extreme compute power — simulations, genomics, weather modeling |
| ETL | Extract, Transform, Load | Pull data from source, clean/reshape it, load into destination |
| ELT | Extract, Load, Transform | Modern approach — load raw data first into data lake, transform later (Glue, dbt) |
| OLTP | Online Transaction Processing | Many small, fast read/write operations — e.g., recording a bank transaction |
| OLAP | Online Analytical Processing | Complex queries over large datasets — e.g., "total claims by region last quarter" |
| NIST | National Institute of Standards and Technology | US government body that publishes security and AI standards |
| OSFI | Office of the Superintendent of Financial Institutions | Canada's banking regulator — sets data residency and security rules for financial institutions |
| PCI DSS | Payment Card Industry Data Security Standard | Security rules for any system that handles credit card data |
| HIPAA | Health Insurance Portability and Accountability Act | US federal law protecting patient health information privacy |
| GDPR | General Data Protection Regulation | EU law giving people rights over their personal data |
| SOC 2 | System and Organization Controls 2 | Security audit certification that proves a company handles customer data responsibly |

---

## ⏱️ 15-MINUTE INTERVIEW NARRATIVE

### Minute 0–2 | Problem Statement
> "We had petabytes of structured transactional data (Oracle, SQL Server) and unstructured content (PDFs, scanned documents, emails, images) sitting on-prem. The business goal was to build an AI platform — specifically a RAG-powered chatbot and autonomous Agentic AI workflows for claims/underwriting/compliance — while ensuring full data governance and audit compliance."

### Minute 2–5 | Migration Strategy
> "We followed a **Hybrid Migration approach**:
> - Files and unstructured blobs → **AWS DataSync** over Direct Connect (encrypted TLS)
> - Databases → **AWS DMS** Full Load + CDC for near-zero downtime
> - One-time large bulk (> 50 TB historical archive) → **Snowball Edge** devices
> - Network backbone → **AWS Direct Connect** (dedicated 10 Gbps) with VPN as failover
> - We landed everything into **S3 Data Lake** with Bronze/Silver/Gold zones using **Glue Catalog** for metadata and **Lake Formation** for fine-grained access control."

### Minute 5–9 | AI Ingestion + RAG Pipeline
> "Once data was in S3:
> - Unstructured PDFs/images → **Textract** (OCR) → normalized JSON → S3 Silver
> - Text content → **Bedrock Embeddings** (Titan or Cohere) → vector store (**OpenSearch Serverless**)
> - Structured data → **Glue ETL** → **Redshift** or **Aurora PostgreSQL with pgvector**
>
> For RAG queries:
> 1. User query → Bedrock Embeddings → vector similarity search (k-NN) in OpenSearch
> 2. Top-K chunks retrieved → injected into prompt context
> 3. Bedrock (Claude) generates grounded, cited response
> 4. Conversation history stored in **DynamoDB**"

### Minute 9–12 | Agentic AI
> "For multi-step autonomous workflows (e.g., claims triage, compliance checks):
> - **Bedrock Agents** orchestrates LLM reasoning + tool use
> - Tools (Action Groups): Lambda functions calling internal APIs, DynamoDB, Redshift
> - **Step Functions** for long-running workflows requiring human approval loops (**A2I**)
> - Agent memory: session state in DynamoDB; long-term memory via Knowledge Bases
> - ReAct pattern: Reason → Act → Observe → Repeat until task complete"

### Minute 12–15 | Governance + Security
> "Non-negotiable for regulated industries:
> - **Bedrock Guardrails**: PII redaction, topic filtering, hallucination controls, deny-list
> - **SageMaker Clarify + Model Monitor**: bias detection, drift alerts, explainability reports
> - **Lake Formation**: column/row-level security — data scientists only see what they're permitted
> - **Macie**: auto-detect PII/PHI in S3 before it enters the AI pipeline
> - **CloudTrail**: every Bedrock API call logged — who invoked what model, with what prompt hash
> - **AWS Config + Security Hub**: continuous compliance against NIST/ISO 27001/OSFI
> - All data encrypted at rest (KMS CMK) and in transit (TLS 1.3)"

---

## 📦 PHASE 1 — DATA MIGRATION

### Migration Decision Tree
```
Data Type?
├── Relational DB (Oracle/SQL Server/MySQL)
│   └── Near-zero downtime needed?
│       ├── YES → DMS Full Load + CDC (Schema Conversion Tool for heterogeneous)
│       └── NO  → DMS Full Load only (maintenance window)
│
├── Files / Unstructured (NFS/SMB/HDFS)
│   └── Volume?
│       ├── < 10 TB  → DataSync over internet (TLS)
│       ├── 10-80 TB → DataSync + Direct Connect (dedicated bandwidth)
│       └── > 80 TB  → Snowball Edge (offline) + DataSync for delta
│
├── Streaming / CDC ongoing
│   └── DMS CDC → Kinesis Data Streams → S3 / Redshift
│
└── VMs / Servers
    └── AWS MGN (Application Migration Service) — lift-and-shift
```

### Migration Tech Stack
| Tool | Purpose | Key Config |
|---|---|---|
| **AWS Direct Connect** | Dedicated 1/10 Gbps private link to AWS | Active + passive VPN for failover |
| **AWS DataSync** | Agent-based NFS/SMB/S3 file transfer | Bandwidth throttling, checksum verify |
| **AWS DMS** | DB migration with CDC | LOB mode for BLOBs; SCT for schema |
| **Snowball Edge 80TB** | Offline petabyte-scale ingest | OpsHub CLI; 10 GbE on-prem copy |
| **AWS Transfer Family** | SFTP/FTPS/FTP managed endpoint to S3 | Identity provider: Cognito / AD |
| **Kinesis Data Firehose** | Stream CDC events to S3/Redshift | Auto-scaling, built-in Parquet conversion |

### Zero-Downtime DB Migration Steps
```
1. SCT       → Convert schema + stored procedures (review conversion report)
2. DMS Task  → Full Load (copy existing rows)
3. DMS Task  → Enable CDC (capture ongoing changes via binlog/redo log)
4. Validate  → Row counts, checksums, application smoke tests
5. Cutover   → Freeze source writes → final CDC flush → update DNS/connection strings
6. Rollback  → Reverse DMS task ready for 48-hr parallel run
7. Decommission → After confidence window
```

---

## 🏗️ PHASE 2 — DATA LAKE ARCHITECTURE (S3)

```
S3 Data Lake Zones:

  BRONZE (Raw / Immutable)
  ├── s3://datalake/raw/databases/     ← DMS output (Parquet/CSV)
  ├── s3://datalake/raw/files/         ← DataSync output (PDF, DOCX, images)
  └── s3://datalake/raw/streams/       ← Kinesis Firehose (JSON events)
  
  SILVER (Curated / Validated)
  ├── s3://datalake/curated/           ← Glue ETL: dedup, schema, PII masked
  └── s3://datalake/curated/vectors/   ← Embeddings ready for vector store
  
  GOLD (AI-Ready / Aggregated)
  ├── s3://datalake/gold/features/     ← Feature Store (SageMaker)
  ├── s3://datalake/gold/redshift/     ← Redshift Spectrum external tables
  └── s3://datalake/gold/knowledge/    ← Bedrock Knowledge Base source

Metadata & Governance:
  AWS Glue Data Catalog   → Schema registry for all tables
  AWS Lake Formation      → Column/row-level access; tag-based permissions
  AWS Macie               → PII/PHI auto-discovery before AI ingestion
  AWS Glue DataBrew       → Visual data profiling + transformation (no-code)
```

---

## 🤖 PHASE 3 — AI INGESTION PIPELINE (Unstructured → Vectors)

### Document Processing Pipeline
```
S3 (PDF/DOCX/Images/Emails)
    │
    ▼
Amazon Textract
    ├── Forms extraction (key-value pairs)
    ├── Table extraction (structured cells)
    └── Raw text (paragraphs, lines)
    │
    ▼
AWS Lambda (Normalize → Chunk)
    ├── Chunking Strategy:
    │   ├── Fixed-size (512 tokens + 10% overlap)       ← Simple, fast
    │   ├── Semantic chunking (sentence boundaries)     ← Better retrieval
    │   └── Hierarchical (parent-child chunks)         ← Best for long docs
    │
    ▼
Amazon Bedrock — Embeddings API
    ├── Titan Text Embeddings v2  (1536 dims, multilingual)
    └── Cohere Embed v3           (1024 dims, domain-specific)
    │
    ▼
Vector Store (choose one)
    ├── Amazon OpenSearch Serverless  ← Managed, k-NN, hybrid search
    ├── Aurora PostgreSQL + pgvector  ← If relational context needed
    ├── Amazon MemoryDB for Redis     ← Ultra-low latency (<1ms)
    └── Pinecone / Weaviate           ← Third-party via Bedrock KB connector

Metadata Index:
    DynamoDB → doc_id, s3_path, chunk_id, source_type, created_at, access_tags
```

### Structured Data → AI Context
```
Relational Data (Aurora / Redshift)
    │
    ▼
AWS Glue ETL → Parquet → S3 Gold
    │
    ├── Redshift → Text-to-SQL (Bedrock Agent action group)
    ├── Aurora   → Direct Lambda query (parameterized SQL)
    └── Athena   → Ad-hoc query from S3 (serverless, pay per query)
```

---

## 💬 PHASE 4a — RAG PIPELINE

### Architecture
```
User Query
    │
    ▼
API Gateway (throttle, auth via Cognito)
    │
    ▼
Lambda Orchestrator
    │
    ├── 1. RETRIEVE
    │       Query → Bedrock Embeddings → OpenSearch k-NN → Top-K chunks
    │       Optional: BM25 keyword search + Semantic → RRF reranking (Cohere Rerank)
    │
    ├── 2. AUGMENT
    │       Prompt = System Prompt + Retrieved Chunks + Conversation History + User Query
    │       History: DynamoDB (last N turns via ConversationBufferWindowMemory)
    │
    └── 3. GENERATE
            Bedrock (Claude 3.5 Sonnet / Titan)
            → Response with source citations
            → Store Q&A pair in DynamoDB for evaluation

Response → API GW → Frontend / Chat UI
```

### RAG Quality Controls
| Technique | Purpose | AWS Service |
|---|---|---|
| Hybrid Search | Keyword + semantic for better recall | OpenSearch hybrid query |
| Reranking | Re-order retrieved chunks by relevance | Cohere Rerank via Bedrock |
| Guardrails | Block irrelevant / harmful responses | Bedrock Guardrails |
| Prompt Cache | Reduce cost for repeated system prompts | Bedrock Prompt Caching |
| Evaluation | RAGAS: faithfulness, relevancy, recall | Lambda + CloudWatch custom metrics |
| Query Routing | Simple → fast model; Complex → Opus | Lambda classifier |

---

## 🧠 PHASE 4b — AGENTIC AI

### Bedrock Agents Architecture
```
User Request (Multi-step task)
    │
    ▼
Bedrock Agent (Orchestrator — Claude)
    │
    ├── ReAct Loop:
    │   Thought → Action → Observation → Thought → ... → Final Answer
    │
    ├── Action Groups (Tools):
    │   ├── Lambda: query_claims_db(claim_id)
    │   ├── Lambda: check_policy_coverage(policy_id, damage_type)
    │   ├── Lambda: trigger_payment_workflow(amount, account)
    │   └── Lambda: search_regulations(jurisdiction, topic)  ← RAG retrieval
    │
    ├── Knowledge Base: RAG retrieval for unstructured context
    │
    └── Memory:
        ├── Session memory (in-context, ephemeral)
        └── Long-term memory (DynamoDB + Bedrock KB)

Human-in-the-Loop:
    Step Functions Wait State + A2I (Augmented AI) → manual review queue
    SQS FIFO → review portal → approve/reject → resume Step Functions task token
```

### Agentic Patterns
```
Pattern              → When to Use                      → AWS Implementation
─────────────────────────────────────────────────────────────────────────────
Tool Use (ReAct)     → Single-agent, multi-step tasks   → Bedrock Agents
Multi-Agent          → Parallel specialized agents      → Bedrock Multi-Agent
Supervisor-Worker    → Manager routes to specialists    → Bedrock Agent + Sub-Agents
Plan-and-Execute     → Complex decomposition upfront    → Step Functions + Bedrock
Reflection           → Self-critique before final answer→ Two-pass Lambda prompt chain
```

---

## 🛡️ PHASE 5 — AI GOVERNANCE

### Governance Framework
```
DATA GOVERNANCE (before AI sees it)
├── Lake Formation    → Column/row-level access; purpose-based tags
├── Macie             → Auto-discover PII/PHI in S3; alert + quarantine
├── Glue Data Quality → Automated data quality rules (completeness, freshness)
└── KMS CMK           → Encrypt all data at rest; key rotation enforced

MODEL GOVERNANCE (during AI inference)
├── Bedrock Guardrails
│   ├── Content Filters   → Block hate, harassment, violence (configurable sensitivity)
│   ├── Topic Deny List   → Block off-topic queries (e.g., competitor mentions)
│   ├── PII Redaction     → Auto-mask before sending to LLM + in response
│   ├── Word Filters      → Custom blocklist / grounding check
│   └── Grounding Check   → Hallucination detection (factual grounding score)
│
├── SageMaker Clarify     → Bias detection in training data + predictions
├── SageMaker Model Monitor → Data drift, model quality drift → CloudWatch alarms
└── Bedrock Model Eval    → Automated eval: ROUGE, BERTScore, custom rubrics

AUDIT & COMPLIANCE (after AI responds)
├── CloudTrail            → Log every Bedrock API call (model ID, prompt hash, latency)
├── AWS Config            → Continuous compliance rules (NIST 800-53, ISO 27001)
├── Security Hub          → Aggregated findings across GuardDuty, Macie, Inspector
├── CloudWatch Logs       → Store prompt + response (for regulated industries)
└── AWS Artifact          → Download SOC2, PCI-DSS, HIPAA compliance reports
```

### AI Governance Interview Talking Points
```
✅ "We implemented purpose limitation — Lake Formation tags define which
   datasets can be used for which AI use cases (e.g., GDPR-restricted
   EU data cannot be used for model training)"

✅ "All LLM inputs/outputs are logged with prompt hash (SHA-256) in
   CloudTrail — regulators can verify no unauthorized prompt injection"

✅ "Bedrock Guardrails PII redaction runs BEFORE the prompt reaches the
   foundation model — the model never sees raw SSN/DOB/account numbers"

✅ "Model drift alerts: SageMaker Model Monitor checks data distribution
   weekly against baseline — if feature drift > threshold, auto-trigger
   retraining pipeline via EventBridge → Step Functions"

✅ "For OSFI (Canada) compliance: all data stays in ca-central-1,
   CloudHSM for key management, Direct Connect for on-prem traffic"
```

---

## 🛠️ COMPLETE TECH STACK SUMMARY

### Migration Layer
| Service | Role |
|---|---|
| AWS Direct Connect (10 Gbps) | Dedicated private network to AWS |
| AWS DataSync | NFS/SMB → S3/EFS file migration with integrity checks |
| AWS DMS + SCT | DB migration + schema conversion (Oracle→Aurora PG) |
| Snowball Edge 80TB | Offline bulk migration > 50 TB |
| Amazon Kinesis Data Firehose | CDC streams → S3 / Redshift |

### Data Lake & Processing
| Service | Role |
|---|---|
| Amazon S3 (Bronze/Silver/Gold) | Central data lake storage |
| AWS Glue (ETL + Catalog) | Serverless ETL, schema registry |
| AWS Lake Formation | Fine-grained data access governance |
| Amazon Redshift + Spectrum | OLAP warehouse + query S3 directly |
| Amazon Athena | Serverless SQL on S3 (ad-hoc analysis) |
| Amazon EMR (Spark) | Large-scale data transformation |

### AI Ingestion
| Service | Role |
|---|---|
| Amazon Textract | OCR for PDFs, forms, tables, handwriting |
| Amazon Comprehend | NLP: entities, PII, sentiment, classification |
| Amazon Bedrock Embeddings | Titan v2 / Cohere Embed v3 vector generation |
| Amazon OpenSearch Serverless | Vector store (k-NN), hybrid search |
| Aurora PostgreSQL + pgvector | Relational vector store for structured context |

### RAG & Agentic AI
| Service | Role |
|---|---|
| Amazon Bedrock (Claude 3.5 Sonnet) | Foundation model for generation |
| Bedrock Knowledge Bases | Managed RAG (ingestion + retrieval) |
| Bedrock Agents | Multi-step agentic orchestration (ReAct) |
| Bedrock Guardrails | Content safety, PII masking, grounding checks |
| AWS Step Functions | Long-running workflow orchestration + human loop |
| Amazon DynamoDB | Conversation history, agent session state, metadata |
| AWS Lambda | Action groups, orchestration glue code |
| Amazon API Gateway | Secure API endpoint (throttle, auth, WAF) |
| Amazon Cognito | User authentication for AI applications |

### AI Governance
| Service | Role |
|---|---|
| Bedrock Guardrails | LLM-level content safety + PII redaction |
| SageMaker Clarify | Bias detection, explainability (SHAP values) |
| SageMaker Model Monitor | Data drift + model quality monitoring |
| Amazon Macie | PII/PHI auto-discovery in S3 |
| AWS CloudTrail | Audit log of all API calls including Bedrock |
| AWS Config | Continuous compliance rule enforcement |
| AWS Security Hub | Aggregated security posture + findings |
| Amazon GuardDuty | ML-based threat detection |
| AWS KMS (CMK) | Encryption key management + rotation |
| AWS CloudHSM | FIPS 140-2 Level 3 (banking/OSFI compliance) |

---

## 🔁 END-TO-END DATA FLOW (One-Pager)

```
ON-PREM
  Oracle DB ──────────────────────────────────────────────────────────┐
  SQL Server ────── DMS (Full Load + CDC) ──────────────────────────┐ │
  NAS/NFS Files ─── DataSync ──────────────────────────────────────┐│ │
  Bulk Archive ──── Snowball Edge ────────────────────────────────┐ ││ │
                                                                   │ ││ │
                     AWS Direct Connect (10 Gbps)                  │ ││ │
                                │                                  │ ││ │
                                ▼                                  │ ││ │
                        S3 BRONZE (Raw)  ◄──────────────────────────┘─┘┘─┘
                                │
                    ┌───────────┴───────────┐
                    │    GLUE ETL + MACIE   │
                    └───────────┬───────────┘
                                │
                        S3 SILVER (Curated)
                                │
             ┌──────────────────┼──────────────────┐
             │                  │                  │
      Textract+Lambda    Glue ETL→Redshift    Kinesis→DynamoDB
      (Unstructured)     (Structured OLAP)    (Events/CDC)
             │
      Bedrock Embeddings
             │
    OpenSearch Serverless
      (Vector Store)
             │
    ┌────────┴──────────────────────────────────┐
    │         RAG + AGENTIC AI RUNTIME           │
    │                                            │
    │  User → API GW → Lambda Orchestrator       │
    │               ├── Bedrock Knowledge Base   │
    │               │   (RAG retrieval)          │
    │               └── Bedrock Agent            │
    │                   (Tool Use / ReAct)       │
    │                   → Action Groups          │
    │                   → Human Review (A2I)     │
    └────────────────────────────────────────────┘
             │
    ┌────────┴──────────────────┐
    │     AI GOVERNANCE         │
    │  Guardrails → Macie       │
    │  CloudTrail → Config      │
    │  Clarify → Model Monitor  │
    └───────────────────────────┘
```

---

## 📚 REFERENCE DOCS — Further Reading

### Official AWS Documentation
| Topic | Link |
|---|---|
| AWS DataSync — User Guide | https://docs.aws.amazon.com/datasync/latest/userguide/what-is-datasync.html |
| AWS DMS — User Guide | https://docs.aws.amazon.com/dms/latest/userguide/Welcome.html |
| AWS Snowball Edge — Developer Guide | https://docs.aws.amazon.com/snowball/latest/developer-guide/whatisedge.html |
| Amazon Bedrock — User Guide | https://docs.aws.amazon.com/bedrock/latest/userguide/what-is-bedrock.html |
| Bedrock Knowledge Bases | https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html |
| Bedrock Agents | https://docs.aws.amazon.com/bedrock/latest/userguide/agents.html |
| Bedrock Guardrails | https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html |
| Amazon OpenSearch Serverless | https://docs.aws.amazon.com/opensearch-service/latest/developerguide/serverless.html |
| SageMaker Clarify | https://docs.aws.amazon.com/sagemaker/latest/dg/clarify-fairness-and-explainability.html |
| SageMaker Model Monitor | https://docs.aws.amazon.com/sagemaker/latest/dg/model-monitor.html |
| AWS Lake Formation | https://docs.aws.amazon.com/lake-formation/latest/dg/what-is-lake-formation.html |
| Amazon Textract | https://docs.aws.amazon.com/textract/latest/dg/what-is.html |

### AWS Whitepapers (Must-Read)
| Whitepaper | Link |
|---|---|
| AWS Well-Architected — Machine Learning Lens | https://docs.aws.amazon.com/wellarchitected/latest/machine-learning-lens/machine-learning-lens.html |
| Migrating to AWS: Best Practices | https://docs.aws.amazon.com/whitepapers/latest/aws-migration-whitepaper/welcome.html |
| Building Data Lakes on AWS | https://docs.aws.amazon.com/whitepapers/latest/building-data-lakes/building-data-lake-aws.html |
| AWS Security Best Practices | https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/welcome.html |
| Generative AI on AWS | https://aws.amazon.com/generative-ai/ |

### AWS Blog Posts (Highly Practical)
| Topic | Link |
|---|---|
| RAG using Bedrock Knowledge Bases | https://aws.amazon.com/blogs/machine-learning/knowledge-base-for-amazon-bedrock/ |
| Agentic AI with Bedrock Agents | https://aws.amazon.com/blogs/aws/build-generative-ai-powered-agents-with-amazon-bedrock/ |
| Implementing Responsible AI with Guardrails | https://aws.amazon.com/blogs/aws/amazon-bedrock-guardrails-now-available/ |
| Hybrid Search OpenSearch + Bedrock | https://aws.amazon.com/blogs/big-data/amazon-opensearch-service-hybrid-query/ |
| Zero-downtime DB Migration with DMS | https://aws.amazon.com/blogs/database/aws-dms-ongoing-replication/ |
| Data Lake with Lake Formation | https://aws.amazon.com/blogs/big-data/build-a-data-lake-foundation-with-aws-glue-and-amazon-s3/ |

### re:Invent Sessions (YouTube — Deep Dives)
| Session | Link |
|---|---|
| Generative AI Best Practices (re:Invent) | https://www.youtube.com/results?search_query=aws+reinvent+generative+ai+bedrock+rag |
| Data Migration at Scale | https://www.youtube.com/results?search_query=aws+reinvent+data+migration+datasync+dms |
| AI Governance on AWS | https://www.youtube.com/results?search_query=aws+reinvent+responsible+ai+governance+bedrock |

### External Authoritative Sources
| Source | Link |
|---|---|
| NIST AI Risk Management Framework | https://www.nist.gov/artificial-intelligence/ai-risk-management-framework |
| EU AI Act Overview | https://artificialintelligenceact.eu/ |
| LangChain AWS Integration Docs | https://python.langchain.com/docs/integrations/providers/aws/ |
| RAGAS — RAG Evaluation Framework | https://docs.ragas.io/en/latest/ |
| Cohere Rerank for AWS | https://docs.cohere.com/docs/reranking |
| pgvector Extension (PostgreSQL) | https://github.com/pgvector/pgvector |

---

## 🧩 ARCHITECTURE ANTI-PATTERNS (Say in Interview)

```
❌ WRONG: Dump all data into one S3 bucket without zones
   ✅ FIX:  Bronze / Silver / Gold with separate IAM + Lake Formation policies

❌ WRONG: Embed PII/PHI in vector store without redaction
   ✅ FIX:  Macie scan → Comprehend PII detection → redact BEFORE embedding

❌ WRONG: Use one large chunk size for all document types
   ✅ FIX:  Tune chunk size per doc type: 256 tokens (FAQs), 1024 tokens (reports)

❌ WRONG: Single Bedrock agent handling all tasks (monolithic agent)
   ✅ FIX:  Supervisor-worker multi-agent: router agent + specialized sub-agents

❌ WRONG: No fallback when vector search returns low-confidence results
   ✅ FIX:  Confidence threshold check → escalate to human / return "I don't know"

❌ WRONG: Running DMS CDC without monitoring replication lag
   ✅ FIX:  CloudWatch metric CDCLatencySource + CDCLatencyTarget → alarm at > 30s

❌ WRONG: Storing conversation history in Lambda memory
   ✅ FIX:  DynamoDB with TTL (auto-expire old sessions to control cost)
```

---

## 📊 COST GUARDRAILS FOR AI PROJECTS

```
Migration Costs:
  DataSync     → $0.0125/GB transferred (first 500 GB free/month)
  DMS          → Instance type cost + data transfer; use r5 for large loads
  Direct Connect → Port fee + data transfer (cheaper than internet at scale)
  Snowball     → $300/job + $15/day beyond 10 days (80 TB device)

AI Runtime Costs:
  Bedrock Claude 3.5 Sonnet → $3/1M input tokens, $15/1M output tokens
  Titan Embeddings v2       → $0.00002/1K tokens
  OpenSearch Serverless     → $0.24/OCU-hour (min 2 OCUs)

Cost Control Tactics:
  ✅ Bedrock Prompt Caching   → Up to 90% cost reduction on repeated system prompts
  ✅ Query routing            → Simple queries → Haiku ($0.25/1M); complex → Sonnet
  ✅ ElastiCache for RAG      → Cache top-1000 frequent queries → zero Bedrock cost
  ✅ S3 Intelligent-Tiering   → Auto-tier historical embeddings/docs to cheaper storage
  ✅ Bedrock Batch Inference  → 50% discount for non-real-time workloads
  ✅ OpenSearch Reserved      → 1-yr reserved capacity vs on-demand
```

---

## ✅ PRE-INTERVIEW CHECKLIST

```
Migration:
□ Can explain DMS Full Load + CDC flow without notes
□ Know when to choose DataSync vs Snowball vs DMS
□ Understand Direct Connect vs VPN tradeoffs

RAG:
□ Can draw RAG pipeline (ingest + query) on whiteboard
□ Know chunking strategies and when to use each
□ Understand hybrid search (BM25 + semantic + RRF reranking)
□ Know Bedrock Knowledge Bases vs custom-built RAG tradeoffs

Agentic AI:
□ Can explain ReAct pattern (Reason → Act → Observe)
□ Know Bedrock Agents architecture (action groups, KB, memory)
□ Understand multi-agent patterns (supervisor-worker)

Governance:
□ Know Bedrock Guardrails capabilities (PII, content, grounding)
□ Can explain how CloudTrail logs Bedrock usage for audit
□ Know Lake Formation column/row-level security model
□ Understand SageMaker Model Monitor drift detection flow

Cost:
□ Know 3 cost optimization tactics for Bedrock
□ Know DataSync pricing model
```

---

---

## 🧰 OSS (Open Source Software) FRAMEWORK LAYER

> **Mentor Note:** AWS gives you the cloud infrastructure (storage, compute, managed databases).
> OSS frameworks give you the engineering control layer that runs ON TOP of that infrastructure.
> Most real projects — including enterprise ones — use both. Do NOT think of these as alternatives;
> think of them as complementary layers. A LangGraph agent can call Bedrock (Claude), store state
> in DynamoDB, retrieve from OpenSearch, and be deployed on ECS Fargate — 100% AWS infra, 100% OSS code.

---

### OSS Tool 1: LangChain — "LEGO Blocks for LLM (Large Language Model) Apps"

**What it is:** LangChain is a Python and JavaScript framework that gives you pre-built, interchangeable
components for every step of building LLM applications. Instead of writing glue code from scratch to
connect an LLM to a vector store to a document loader, you pick LangChain components and assemble them.

**Simple analogy:** Building a car. You could machine every part yourself (raw Boto3/Bedrock SDK calls)
or buy standard pre-made parts (LangChain components) and assemble them. LangChain provides the parts.

```
LangChain Component     →  What it does                      →  AWS Native Equivalent
────────────────────────────────────────────────────────────────────────────────────────
Document Loaders        →  Load PDFs, web pages, DBs, email  →  Textract + custom Lambda
Text Splitters          →  Chunk documents intelligently     →  Custom Lambda chunking code
Embedding Models        →  Convert text → vectors            →  Bedrock Embeddings API
Vector Stores           →  Store + search vectors            →  OpenSearch Serverless
Chains                  →  Connect steps in a pipeline       →  Step Functions (simpler cases)
Agents                  →  LLM + tools in a reasoning loop   →  Bedrock Agents
Memory                  →  Conversation history management   →  DynamoDB session store
Callbacks               →  Log and trace every step          →  CloudWatch + X-Ray
```

**When to use LangChain:**
```
POC / Demo in hours          → YES — fastest path to a working RAG demo
Document Q&A over PDFs       → YES — excellent loaders and chain types
Simple tool-use agent        → YES — 2-3 tools works very well
Complex branching agent      → NO  → use LangGraph (see next section)
Best retrieval quality       → NO  → use LlamaIndex (better indexing options)
```

**LangChain Strengths:**
```
✅ 100+ document loaders: PDF, Word, web, SQL, email, Notion, Confluence
✅ Works with ANY LLM: swap Bedrock for OpenAI or Ollama with one line change
✅ Huge community: nearly every problem has a Stack Overflow answer
✅ Fastest POC path: working RAG demo in under 2 hours
✅ Portable: same code runs locally, on EC2, on ECS, on Lambda container
```

**LangChain Limitations:**
```
❌ Multiple breaking versions (v0.1 → v0.2 → v0.3) changed APIs frequently
❌ For complex agents, the abstraction fights you — hard to trace what is happening
❌ Not ideal for agents with loops, retries, or human-in-the-loop → use LangGraph
❌ Python performance overhead — not suited for sub-100ms latency requirements
```

**LangChain RAG (Retrieval-Augmented Generation) Code Example — with Comments:**
```python
# -----------------------------------------------------------------------
# LangChain RAG Pipeline on AWS
# Deployed on: AWS Lambda (container image) or EC2
# AWS Services used: S3, Bedrock, OpenSearch Serverless
# -----------------------------------------------------------------------

from langchain_aws import BedrockEmbeddings, ChatBedrock
# BedrockEmbeddings: calls AWS Bedrock to convert text into vectors (lists of numbers)
# ChatBedrock: wrapper to call Claude, Titan, or other Bedrock models as a chat LLM

from langchain_community.document_loaders import S3FileLoader
# S3FileLoader: downloads a file from S3 and gives back LangChain Document objects
# A Document object has two fields: .page_content (the text) and .metadata (source info)

from langchain.text_splitter import RecursiveCharacterTextSplitter
# RecursiveCharacterTextSplitter: splits text trying to keep logical units together
# First tries to split by paragraphs, then sentences, then characters
# Much smarter than splitting at a fixed character count

from langchain_community.vectorstores import OpenSearchVectorSearch
# OpenSearchVectorSearch: LangChain wrapper for AWS OpenSearch
# Handles creating the index, storing vectors, and running k-NN (k-Nearest Neighbors) search

from langchain.chains import RetrievalQA
# RetrievalQA: a pre-built chain that wires together: retrieve → augment → generate
# You configure it once and call it like a function

# STEP 1: Load a PDF from S3 (Simple Storage Service) Silver zone
loader = S3FileLoader(
    bucket="my-datalake-silver",       # the S3 bucket name
    key="policies/home_policy_2026.pdf" # path to the file inside the bucket
)
documents = loader.load()
# documents is a list of Document objects — one per page of the PDF

# STEP 2: Split the PDF pages into smaller chunks
# Why chunk? LLMs have token limits. Smaller chunks = more precise retrieval.
splitter = RecursiveCharacterTextSplitter(
    chunk_size=512,   # each chunk is at most 512 characters
    chunk_overlap=50  # 50-char overlap between chunks so we do not cut a sentence in half
)
chunks = splitter.split_documents(documents)
# A 30-page policy PDF might become 180 chunks

# STEP 3: Convert chunks to vectors and store in OpenSearch
embeddings = BedrockEmbeddings(
    model_id="amazon.titan-embed-text-v2:0", # AWS Titan Embeddings model — 1536 dimensions
    region_name="ca-central-1"              # keep data in Canada for OSFI compliance
)

# This single call does two things:
# 1. Sends each chunk to Bedrock → gets back a vector (1536 numbers per chunk)
# 2. Stores all vectors in OpenSearch Serverless for future search
vectorstore = OpenSearchVectorSearch.from_documents(
    documents=chunks,
    embedding=embeddings,
    opensearch_url="https://your-collection.ca-central-1.aoss.amazonaws.com",
    index_name="home-policy-index"  # like a table name in the vector database
)

# STEP 4: Build the RAG chain using Claude via Bedrock
llm = ChatBedrock(
    model_id="anthropic.claude-3-5-sonnet-20241022-v2:0",
    region_name="ca-central-1",
    model_kwargs={
        "max_tokens": 1024,
        "temperature": 0  # 0 = deterministic answers, no randomness — best for factual Q&A
    }
)

qa_chain = RetrievalQA.from_chain_type(
    llm=llm,
    retriever=vectorstore.as_retriever(
        search_kwargs={"k": 4}  # fetch the 4 most similar chunks per query
        # Why 4? Enough context without overloading the prompt. Tune based on chunk size.
    ),
    return_source_documents=True  # return which chunks were used so you can show citations
)

# STEP 5: Answer a question
result = qa_chain.invoke({"query": "What is the deductible for basement flooding?"})
print(result["result"])            # Claude's answer grounded in the policy
print(result["source_documents"]) # which policy sections supported the answer
```

---

### OSS Tool 2: LangGraph — "State Machines for AI Agents"

**What it is:** LangGraph is a library (built on top of LangChain) for building stateful, multi-step
AI agents where the workflow is a GRAPH (nodes and edges) rather than a linear chain. The critical
difference: LangGraph supports CYCLES. An agent can loop, retry, branch, and wait for human input.

**Simple analogy:** LangChain is a straight pipeline (A → B → C). LangGraph is a flowchart
(A → B → if condition → go to C or loop back to A). Real agents need flowcharts, not pipelines.

**Why this matters:** Bedrock Agents is a black box — AWS runs it, you configure it via console or
API. LangGraph gives you the SAME capabilities in code you own, can test, can version in Git, and
can deploy anywhere. This portability is why enterprises with strict compliance requirements prefer it.

```
LangGraph Core Concepts — Plain English:
────────────────────────────────────────────────────────────────────────────────────
StateGraph    → The blueprint for your entire agent workflow
State         → A shared dictionary passed between every step — like a shared whiteboard
Node          → One function in the workflow (call an LLM, query a DB, call an API)
Edge          → A connection between two nodes (can be conditional — "go here IF this")
Checkpointer  → Saves the state to a database after each node completes
               → This is HOW human-in-the-loop works: pause, save, human decides, resume
```

**LangGraph Insurance Claims Agent — with Comments:**
```python
# -----------------------------------------------------------------------
# LangGraph Multi-Step Agent: Insurance Claims Triage
# Deployed on: AWS ECS (Elastic Container Service) Fargate
# Why ECS not Lambda? Agents can run > 15 minutes (Lambda's hard limit)
# AWS Services: Bedrock (Claude), DynamoDB (state), SQS (human review queue)
# -----------------------------------------------------------------------

from langgraph.graph import StateGraph, END
# StateGraph: the main class — you add nodes and edges to build your workflow
# END: a special sentinel that tells the graph "this path is done"

from langgraph.checkpoint.dynamodb import DynamoDBSaver
# DynamoDBSaver: after EACH node completes, saves the full state to DynamoDB
# This is what makes pause-and-resume possible (human-in-the-loop)

from langchain_aws import ChatBedrock
from typing import TypedDict, List, Optional

# STEP 1: Define the State (the shared whiteboard all nodes read from and write to)
class ClaimState(TypedDict):
    claim_id: str            # which claim is being processed
    claim_text: str          # raw text of the claim submitted by customer
    extracted_data: dict     # structured fields extracted by Claude (amount, date, type)
    fraud_score: float       # probability of fraud from 0.0 (clean) to 1.0 (very suspicious)
    decision: str            # final outcome: "APPROVED", "REJECTED", or "PENDING_HUMAN_REVIEW"

# STEP 2: Define Nodes (each is a Python function that reads state and returns updated state)

def extract_claim_details(state: ClaimState) -> ClaimState:
    # Node 1: Ask Claude to extract structured fields from the raw claim text
    # Example input: "My basement flooded May 10th, I estimate $8000 damage to furniture"
    # Example output: {"incident_type": "flood", "date": "2026-05-10", "amount": 8000}
    llm = ChatBedrock(model_id="anthropic.claude-3-5-sonnet-20241022-v2:0")
    prompt = f"""Extract the following fields from this insurance claim as JSON:
    - incident_type (e.g. flood, fire, theft)
    - incident_date (YYYY-MM-DD format)
    - claimed_amount (number only)
    
    Claim text: {state['claim_text']}"""
    
    response = llm.invoke(prompt)
    state["extracted_data"] = response.content
    return state  # return state with extracted_data filled in — next node will see this

def check_fraud_risk(state: ClaimState) -> ClaimState:
    # Node 2: Call AWS Fraud Detector ML model to score this claim
    # Fraud Detector is a trained model that scores based on historical fraud patterns
    import boto3, json
    client = boto3.client("frauddetector", region_name="ca-central-1")
    
    try:
        response = client.get_event_prediction(
            detectorId="insurance-claims-detector",
            eventId=state["claim_id"],
            eventTypeName="claim-submission",
            eventVariables={
                "claimed_amount": str(state["extracted_data"].get("claimed_amount", 0)),
                "incident_type": state["extracted_data"].get("incident_type", "unknown")
            }
        )
        # Map Fraud Detector outcomes to a simple 0.0-1.0 score
        outcomes = response["ruleResults"][0]["outcomes"]
        state["fraud_score"] = 0.85 if "HIGH_RISK" in outcomes else (0.5 if "MEDIUM_RISK" in outcomes else 0.1)
    except Exception:
        state["fraud_score"] = 0.5  # if Fraud Detector call fails, default to medium risk
    
    return state

def route_decision(state: ClaimState) -> str:
    # Conditional Edge (Router): this function is called after fraud_check
    # It returns the NAME of the next node to go to — this is what makes branching work
    if state["fraud_score"] >= 0.5:
        return "send_to_human"   # medium or high risk → human must review
    else:
        return "auto_approve"    # low risk → safe to auto-approve

def send_to_human_review(state: ClaimState) -> ClaimState:
    # Node 3a: Put this claim in the human review queue and pause
    # The DynamoDBSaver checkpointer saves the full state here
    # A claims officer in the review portal will see this claim and click Approve/Reject
    # Their action calls an external API that resumes this graph with the human decision
    import boto3
    sqs = boto3.client("sqs", region_name="ca-central-1")
    sqs.send_message(
        QueueUrl="https://sqs.ca-central-1.amazonaws.com/123456789/claims-review-queue",
        MessageBody=f'{{"claim_id": "{state["claim_id"]}", "fraud_score": {state["fraud_score"]}}}',
        MessageGroupId=state["claim_id"]  # FIFO (First In, First Out) queue grouping
    )
    state["decision"] = "PENDING_HUMAN_REVIEW"
    return state  # graph pauses here — resumes when human reviewer responds

def auto_approve(state: ClaimState) -> ClaimState:
    # Node 3b: Low-risk claim — approve automatically and trigger payment
    state["decision"] = "APPROVED"
    import boto3
    sfn = boto3.client("stepfunctions", region_name="ca-central-1")
    sfn.start_execution(
        stateMachineArn="arn:aws:states:ca-central-1:123456789:stateMachine:payment-workflow",
        input=f'{{"claim_id": "{state["claim_id"]}"}}'
        # Kick off the separate payment Step Functions workflow
    )
    return state

# STEP 3: Build the Graph (wire all nodes and edges together)
workflow = StateGraph(ClaimState)

# Register every node by name
workflow.add_node("extract", extract_claim_details)
workflow.add_node("fraud_check", check_fraud_risk)
workflow.add_node("send_to_human", send_to_human_review)
workflow.add_node("auto_approve", auto_approve)

# Define flow:
workflow.set_entry_point("extract")              # always start at extract
workflow.add_edge("extract", "fraud_check")      # extract → always → fraud_check
workflow.add_conditional_edges(                  # fraud_check → route based on score
    "fraud_check",
    route_decision,                              # this function chooses the next node
    {"send_to_human": "send_to_human", "auto_approve": "auto_approve"}
)
workflow.add_edge("send_to_human", END)          # after human review queued → done
workflow.add_edge("auto_approve", END)           # after auto approve → done

# STEP 4: Add checkpointing so state is saved to DynamoDB after each node
# This is what makes human-in-the-loop possible — state survives across days
checkpointer = DynamoDBSaver(table_name="langgraph-state", region_name="ca-central-1")
app = workflow.compile(checkpointer=checkpointer)

# STEP 5: Run the agent
result = app.invoke(
    input={
        "claim_id": "CLM-2026-00456",
        "claim_text": "Basement flooded on May 15 2026 after heavy rain. Damage to appliances $12,000.",
        "extracted_data": {},
        "fraud_score": 0.0,
        "decision": ""
    },
    config={"configurable": {"thread_id": "CLM-2026-00456"}}
    # thread_id: unique identifier for this workflow run — used by checkpointer to save/restore
)
print(result["decision"])  # "APPROVED" or "PENDING_HUMAN_REVIEW"
```

**LangGraph vs Bedrock Agents — Direct Comparison:**
```
Dimension              LangGraph                        Bedrock Agents
──────────────────────────────────────────────────────────────────────────
Control                You own the code fully           AWS black box — limited visibility
Debugging              Full Python stack traces         Limited CloudWatch logs
Cycles / Loops         Yes — core feature               No — linear tool call sequences only
Human-in-the-loop      Yes — built-in checkpointing     Limited — requires workarounds
Multi-agent            Yes — supervisor/worker in code  Yes — but less flexible
Deployment             You deploy (ECS, EC2, K8s)       Serverless — AWS manages it
Cost                   ECS compute cost                 Per API call pricing
Portability            Runs anywhere Python runs        AWS only
Best for               Complex, production-grade agents Quick POC, simple tool use
Setup time             More code to write               Faster for simple agents
```

---

### OSS Tool 3: LangSmith — "Observability for LLM Applications"

**What it is:** LangSmith is a platform (by the LangChain team) that gives you full visibility into
every step of your LLM application — every prompt, every retrieval result, every LLM response,
every tool call — with timing, token counts, and cost.

**Simple analogy:** If CloudWatch is your application health monitor, LangSmith is your LLM debugger.
When your RAG pipeline gives a wrong answer, LangSmith lets you click into that exact run and see:
which chunks were retrieved (and their similarity scores), what the full prompt looked like, what
Claude returned, how long each step took, and exactly how much it cost in tokens.

**Why you need it:** Without it, debugging a bad RAG response is guesswork.

```python
# -----------------------------------------------------------------------
# LangSmith Setup — Zero code changes needed in your pipeline
# Add these environment variables and ALL LangChain/LangGraph calls are traced
# In AWS: store in Secrets Manager, inject into ECS task definition as env vars
# -----------------------------------------------------------------------

import os

os.environ["LANGCHAIN_TRACING_V2"] = "true"
# Tells LangChain to send every call to LangSmith — traces everything automatically

os.environ["LANGCHAIN_API_KEY"] = "ls__your_key_here"
# Your LangSmith API (Application Programming Interface) key from app.smith.langchain.com

os.environ["LANGCHAIN_PROJECT"] = "insurance-rag-production"
# Groups all traces under this project name in LangSmith UI

# That is all. Your existing pipeline code is unchanged.
# LangSmith now captures for EVERY run:
#   - The user's exact question
#   - Which document chunks were retrieved (with similarity scores)
#   - The full prompt sent to Claude (system message + context + question)
#   - Claude's exact response
#   - Time taken at each step
#   - Input and output token counts (= cost)
#   - Any errors with full stack traces

# -----------------------------------------------------------------------
# LangSmith Evaluation — Test your RAG quality automatically
# -----------------------------------------------------------------------
from langsmith import Client
from langsmith.evaluation import evaluate

client = Client()

# Create a ground truth dataset — pairs of (question, correct answer)
# This is your benchmark test suite — run it every time you change the pipeline
dataset = client.create_dataset(
    dataset_name="insurance-policy-qa-benchmark",
    description="50 questions with verified correct answers from policy documents"
)
client.create_examples(
    inputs=[
        {"query": "What is the flood deductible for basement damage?"},
        {"query": "Is earthquake damage covered under the standard policy?"},
    ],
    outputs=[
        {"answer": "The basement flood deductible is $2,500"},
        {"answer": "Earthquake damage is NOT covered. A separate earthquake rider is required."},
    ],
    dataset_id=dataset.id
)

# Run evaluation: LangSmith sends each question through your pipeline
# and scores the responses against the expected answers
results = evaluate(
    lambda inputs: qa_chain.invoke(inputs),  # your RAG chain
    data=dataset,
    evaluators=["qa"],           # built-in evaluator: is the answer correct?
    experiment_prefix="rag-v2"   # compare this run against "rag-v1" to see improvement
)
# You can now see: 82% of answers correct (up from 71% in v1), which ones failed, and why
```

**LangSmith vs AWS CloudWatch for LLM Debugging:**
```
Problem                          LangSmith                    CloudWatch + X-Ray
────────────────────────────────────────────────────────────────────────────────
"Why did RAG return wrong answer?" Click into the trace        Write custom log parsing queries
See retrieved chunks + scores    Built-in, visual             Log manually, no structure
Full prompt inspection           One click                    Search through raw log strings
Compare v1 vs v2 of prompt       Side-by-side experiment UI   Build custom dashboards
Token cost per request           Automatic                    Build custom metric filters
Eval against ground truth        Built-in eval framework      Build from scratch
Data stays in AWS                NO (goes to LangSmith cloud) YES — stays in your account
Regulated industry safe          Self-hosted version needed   Yes — AWS native
```

---

### OSS Tool 4: LlamaIndex — "The Best Way to Get Your Data INTO an LLM"

**What it is:** LlamaIndex (formerly GPT Index) is a data framework that specializes in connecting
LLMs to your data. While LangChain is a general-purpose LLM framework, LlamaIndex focuses entirely
on the "get the right data into the LLM" problem — and it does it with more sophisticated options
than LangChain or Bedrock Knowledge Bases.

**Simple analogy:** LangChain is a Swiss Army knife — does everything adequately. LlamaIndex is a
specialist scalpel for data retrieval — fewer features, but better at its specific job.

**When to switch from LangChain to LlamaIndex:** When your RAG quality is poor (wrong chunks
retrieved), LlamaIndex's richer indexing and query engine options usually improve it.

```
LlamaIndex Key Components — Plain English:
────────────────────────────────────────────────────────────────────────────────
Data Connectors (SimpleDirectoryReader, S3Reader)
  → 150+ loaders: PDF, Word, PowerPoint, CSV, SQL databases, APIs, web pages
  → More formats and better extraction than LangChain loaders

Node Parsers (how documents are chunked)
  → SemanticSplitterNodeParser: splits on MEANING boundaries, not characters
    Better than fixed-size: a paragraph about deductibles stays together
  → HierarchicalNodeParser: creates parent chunks and child chunks
    Retrieval finds child chunk (precise), but LLM receives parent chunk (more context)
  → SentenceWindowNodeParser: retrieves a sentence but gives LLM surrounding sentences

Index Types (how chunks are stored and searched)
  → VectorStoreIndex: standard vector similarity search (same as LangChain)
  → SummaryIndex: summarize ALL documents first, answer from summaries
  → KnowledgeGraphIndex: extract entities and relationships, build a graph
    Good for: "How is Product A related to Regulation B?"
  → SQLStructStoreIndex: natural language → SQL query → structured answer
    Good for: mixing database queries with document search

Query Engines (how questions are answered)
  → VectorIndexQueryEngine: standard RAG retrieval
  → RouterQueryEngine: decides WHICH index to search based on the question
    "What is the deductible?" → searches policy docs
    "How many claims last month?" → queries Redshift
  → SubQuestionQueryEngine: breaks a complex question into sub-questions
    "Compare flood vs fire deductibles" → two sub-queries → one merged answer
```

**LlamaIndex vs LangChain for RAG — When to Choose:**
```
Situation                          LlamaIndex   LangChain    Why
──────────────────────────────────────────────────────────────────────────────────
Simple RAG on < 100 documents      ✓            ✓ (easier)   LangChain faster setup
RAG on 10,000+ documents           ✓ (better)   ✓            LlamaIndex better indexing
Mixed SQL + document search        ✓ (built-in) ✗ (manual)   SQLJoinQueryEngine
Retrieval quality is poor          ✓ (fix it)   ✗            Better chunking options
Already using LangChain            ✓ (works)    ✓            LlamaIndex integrates with LC
Agentic workflows, tool use        ✗            ✓ LangGraph  Not LlamaIndex's strength
```

---

### OSS Tool 5: MLflow — "Git for Machine Learning Experiments"

**What it is:** MLflow is an open-source platform for managing the machine learning lifecycle.
It tracks experiments (what parameters you used, what results you got), versions models, and
can serve models as REST (Representational State Transfer) API endpoints.

**Simple analogy:** You run 50 experiments training a fraud detection model, tweaking the number
of trees, the learning rate, the sampling strategy. Without MLflow, you have 50 folders of output
files and no way to remember what worked. With MLflow, every run is logged — you can sort by
accuracy and see exactly what settings produced the best model.

```
MLflow Components:
────────────────────────────────────────────────────────
MLflow Tracking  → Log: parameters (inputs), metrics (results), artifacts (model files)
MLflow Projects  → Package code + dependencies so anyone can reproduce your experiment
MLflow Models    → Standard format to save/load models across frameworks (sklearn, PyTorch, etc.)
MLflow Registry  → Version models: Staging → Production → Archived workflow
MLflow Serving   → Deploy registered models as REST API endpoints

AWS Deployment:
  MLflow Tracking Server → EC2 instance, backed by:
    - RDS (Relational Database Service) PostgreSQL: stores experiment metadata
    - S3 (Simple Storage Service): stores model artifacts (the actual model files)
  Alternative: Amazon SageMaker Experiments (AWS native equivalent — less flexible, no portability)
```

```python
# MLflow experiment tracking — running on EC2 with RDS + S3 backend
import mlflow
import mlflow.sklearn
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import f1_score

# Point MLflow at your EC2-hosted tracking server
# In AWS: EC2 runs mlflow server, backed by RDS for metadata, S3 for artifacts
mlflow.set_tracking_uri("http://mlflow-server.internal:5000")
mlflow.set_experiment("fraud-detection-v2")
# All runs inside this 'with' block are grouped under this experiment name

with mlflow.start_run(run_name="random-forest-200trees-depth10"):
    # Log parameters — the recipe: what settings you used
    mlflow.log_param("n_estimators", 200)     # number of decision trees in the forest
    mlflow.log_param("max_depth", 10)          # how deep each tree is allowed to grow
    mlflow.log_param("class_weight", "balanced")
    # class_weight=balanced: important for fraud data where fraud cases are rare (imbalanced)

    model = RandomForestClassifier(n_estimators=200, max_depth=10, class_weight="balanced")
    model.fit(X_train, y_train)

    # Log metrics — the taste test: how well did it perform?
    accuracy = model.score(X_test, y_test)
    f1 = f1_score(y_test, model.predict(X_test))
    # F1 score = balance between precision and recall
    # Better metric than accuracy for fraud detection because fraud cases are rare
    mlflow.log_metric("accuracy", accuracy)
    mlflow.log_metric("f1_score", f1)

    # Save the model itself to S3 via MLflow
    mlflow.sklearn.log_model(model, "fraud-detection-model")
    # Now in the MLflow UI you can compare this run against all previous runs,
    # sort by f1_score, and register the best one as "Production"
```

---

### OSS Tool 6: Apache Airflow — "Cron Jobs with Superpowers"

**What it is:** Apache Airflow is the industry standard open-source workflow orchestration platform.
You define workflows as Python code in the form of DAGs (Directed Acyclic Graphs — workflows that
flow forward without loops). Airflow manages scheduling, retries, alerting, and logging for you.

**Simple analogy:** Imagine you have a 10-step data pipeline. Without Airflow, you set up 10 cron
jobs and hope they run in order and don't fail silently. With Airflow, you define the steps and
dependencies once. If step 3 fails, steps 4-10 do not run, you get an email, and you can retry step
3 with one click from a visual dashboard.

```
Airflow on AWS — Deployment Options:
────────────────────────────────────────────────────────────────────────────────
Amazon MWAA (Managed Workflows for Apache Airflow) → AWS runs Airflow for you
  - No server management
  - Integrates with IAM (Identity and Access Management), VPC, S3
  - More expensive than self-hosted
  - Best for teams that want Airflow without the ops overhead

Self-hosted on EKS (Elastic Kubernetes Service)
  - Full control
  - Cheaper at large scale
  - You manage upgrades, scaling, storage
  - Best for platform engineering teams with Kubernetes expertise

AWS Native Alternative: Step Functions
  - Better for event-driven, serverless workflows
  - Less powerful for complex data engineering DAGs
  - No Python — workflows defined in Amazon States Language (JSON)
  - Choose Airflow when: data engineering team already knows it, complex dependencies
  - Choose Step Functions when: serverless, event-driven, simpler flows
```

---

### OSS Tool 7: dbt (data build tool) — "Version-Controlled SQL Transformations"

**What it is:** dbt lets you transform data in your warehouse by writing SQL SELECT statements.
dbt handles everything else: creating tables and views, running data quality tests, generating
documentation, and visualizing data lineage (how data flows from source to final table).

**Simple analogy:** Normally you write a SQL script in a text file, run it manually, and hope
the output is correct. dbt turns those SQL files into a proper software engineering project —
with version control (Git), automated tests, auto-generated docs, and a visual lineage graph.

```
dbt Concepts — Plain English:
────────────────────────────────────────────────────────────────────────────────
Model         → A .sql file containing a SELECT statement
               → dbt runs it and creates a table or view in Redshift/Athena/Aurora
Materialization → How to persist the output:
               → 'table': create a full table (run every time)
               → 'view': create a virtual view (no data stored, query on demand)
               → 'incremental': only process new rows (much faster for large tables)
Test          → Assertions about your data: no nulls, unique values, valid references
Source        → Declare your raw tables (from DMS, DataSync, Kinesis) as inputs
Lineage       → Auto-generated graph: click any table and see where it came from

AWS Integration:
  dbt Core (free, OSS) → runs as a scheduled ECS task or Lambda; targets Redshift or Athena
  dbt Cloud (paid SaaS) → has built-in scheduler, browser IDE, CI/CD — easiest to start with
```

```sql
-- -----------------------------------------------------------------------
-- dbt model: models/silver/stg_insurance_claims.sql
-- Purpose: Transform raw Oracle claims data (Bronze S3 via DMS) into
--          a clean, validated, PII-masked Silver layer for AI ingestion
-- -----------------------------------------------------------------------

{{ config(
    materialized='incremental',  -- only process rows newer than last run
    unique_key='claim_id',       -- if claim_id exists, UPDATE it instead of INSERT
    schema='silver'              -- create this table in the 'silver' Redshift schema
) }}
-- The {{ }} syntax is Jinja templating — dbt processes this before running the SQL

SELECT
    TRIM(CAST(claim_id AS VARCHAR))          AS claim_id,
    -- TRIM removes leading/trailing spaces — Oracle exports often include them
    -- CAST ensures the type is VARCHAR even if source has NUMBER type

    TO_DATE(claim_date, 'DD-MON-YYYY')       AS claim_date,
    -- Oracle stores dates as 'DD-MON-YYYY' (e.g., '15-MAY-2026')
    -- We standardize to DATE type for consistency across all downstream models

    UPPER(TRIM(claim_status))                AS claim_status,
    -- Source has mixed casing: "approved", "APPROVED", "Approved" — normalize to uppercase

    '****' || RIGHT(account_number, 4)       AS masked_account_number,
    -- PII (Personally Identifiable Information) masking: keep only last 4 digits
    -- The full account number NEVER enters the Silver layer or AI pipeline

    COALESCE(claim_amount_cad, 0)            AS claim_amount_cad,
    -- COALESCE replaces NULL with 0 — avoids NULL propagation errors downstream

    CURRENT_TIMESTAMP                        AS dbt_processed_at,
    -- Audit column: always know when this row was last processed
    '{{ this }}'                             AS dbt_source_model
    -- {{ this }} = this model's own table name — useful for data lineage tracking

FROM {{ source('bronze', 'raw_claims_oracle') }}
-- {{ source(...) }} declares this comes from a raw source (DMS output in Bronze layer)
-- dbt generates a visual lineage diagram from all {{ source }} and {{ ref }} references

{% if is_incremental() %}
-- For incremental runs: only grab rows that arrived AFTER the last successful run
-- This makes daily runs fast even with millions of historical records
WHERE claim_date > (SELECT MAX(claim_date) FROM {{ this }})
{% endif %}
```

---

### OSS Tool 8: Hugging Face — "GitHub for AI Models"

**What it is:** Hugging Face is a platform and library ecosystem for AI. It hosts 300,000+
pre-trained open-source models you can download and run yourself — no API key, no per-token
cost, full control. The `transformers` library lets you load any of these models in Python.

**Why it matters for your project:** Not every AI problem needs Claude or GPT-4. For document
classification, named entity recognition, PII detection, or creating text embeddings, a smaller
open-source model running on your own EC2 GPU instance can be:
- **Cheaper**: no per-token cost once the model is loaded
- **Faster**: no network round-trip to Bedrock
- **More private**: data never leaves your VPC (Virtual Private Cloud)
- **Customizable**: fine-tune on your own domain data using LoRA (Low-Rank Adaptation)

```
Key Hugging Face Libraries:
────────────────────────────────────────────────────────────────────────────────
transformers         → Load and run any pre-trained model (BERT, LLaMA, Mistral, etc.)
sentence-transformers → Best library for text embeddings — fine-tune on your domain data
datasets             → Load and preprocess training datasets
peft                 → Parameter-Efficient Fine-Tuning — fine-tune LLMs cheaply (LoRA/QLoRA)
accelerate           → Automatically distribute training across multiple GPUs

AWS Deployment Options:
  EC2 GPU instances (g4dn, g5, p3) → Run models directly — full control, pay hourly
  SageMaker Real-time Endpoints    → Deploy as managed auto-scaling API endpoint
  SageMaker JumpStart              → One-click deploy of popular HuggingFace models
  AWS Inferentia / Trainium chips  → AWS custom AI chips — 40-70% cheaper inference than GPU
```

```python
# -----------------------------------------------------------------------
# Hugging Face sentence-transformers for custom domain embeddings
# Running on: EC2 g4dn.xlarge (1x NVIDIA T4 GPU — ~$0.53/hour)
# Or: SageMaker Endpoint for production auto-scaling
#
# Why use this instead of Bedrock Titan Embeddings?
# - Can fine-tune on YOUR insurance/banking domain vocabulary
# - No per-token cost: embed 1 million chunks for ~$0.50 (GPU time)
#   vs Bedrock Titan: ~$20 for 1 million chunks
# - Data stays in your VPC (Virtual Private Cloud) — never leaves AWS
# -----------------------------------------------------------------------

from sentence_transformers import SentenceTransformer

# Load a pre-trained embedding model from Hugging Face Hub
# This downloads ~1.3 GB on first run, then caches locally
# BAAI = Beijing Academy of Artificial Intelligence
# BGE = BAAI General Embedding — one of the best open-source models (2024-2025)
model = SentenceTransformer(
    "BAAI/bge-large-en-v1.5",
    # bge-large: 1024 dimensions, excellent English performance
    # Alternative for multilingual: "BAAI/bge-m3" (supports 100+ languages)
    device="cuda"   # use GPU — falls back to CPU if no GPU available
)

# Generate embeddings for a batch of document chunks
# These replace Bedrock Titan Embeddings in your pipeline — everything else stays the same
policy_chunks = [
    "Flood damage to finished basements requires a $2,500 deductible.",
    "Earthquake and land movement damage is excluded from standard coverage.",
    "Fire damage is covered up to full replacement value of the structure."
]

vectors = model.encode(
    policy_chunks,
    batch_size=64,              # process 64 chunks at a time — adjust based on GPU memory
    show_progress_bar=True,
    normalize_embeddings=True   # normalize to length 1 — makes cosine similarity much faster
)
# vectors.shape = (3, 1024)
# Each chunk is now a list of 1024 numbers representing its semantic meaning
# Chunks about similar topics will have vectors that are mathematically close together

# Store in OpenSearch exactly as before — your RAG pipeline is unchanged
# Only the embedding generation step changed (HuggingFace instead of Bedrock)
```

---

---

## ⚖️ NATIVE AWS vs OSS — Full Comparison by Use Case

> **Mentor Note:** This is the most important section for interviews. Senior architects are expected
> to know BOTH worlds and explain WHEN to choose which. The answer is never "always use AWS native"
> or "always use OSS." The answer depends on control needs, team skills, compliance, and timeline.

### Use Case 1: LLM Agent Orchestration
| Dimension | Bedrock Agents (AWS Native) | LangGraph (OSS on AWS) |
|---|---|---|
| **What it is** | Fully managed agent service — AWS handles the runtime | Python library you deploy yourself on ECS/EC2 |
| **Setup speed** | Fast — configure via console or API | Slower — write Python code, deploy container |
| **Control** | Low — AWS decides internal loop logic | High — you control every node and edge |
| **Debugging** | CloudWatch logs (limited LLM context) | LangSmith traces (full prompt/response visibility) |
| **Cycles/Loops** | Not supported — linear tool call only | Fully supported — agents can retry and branch |
| **Human-in-loop** | Workaround needed | Native — checkpointer pauses/resumes graph |
| **Multi-agent** | Supported but limited configurability | Full control — supervisor/worker in plain Python |
| **Cost** | Per API call (pay per use) | ECS compute cost (pay per hour) |
| **Portability** | AWS only | Runs on any cloud or on-prem |
| **Best for** | Simple agents, quick POC, AWS-only shops | Complex agents, regulated env, multi-cloud |
| **❌ Limitation** | Black box — cannot inspect internal reasoning | You manage deployment, scaling, infrastructure |

### Use Case 2: RAG (Retrieval-Augmented Generation) Pipeline
| Dimension | Bedrock Knowledge Bases (AWS Native) | LlamaIndex + OpenSearch (OSS on AWS) |
|---|---|---|
| **What it is** | Fully managed RAG — AWS handles ingestion and retrieval | Python framework you run on EC2 or Lambda |
| **Setup speed** | Fast — point to S3, choose embedding model, done | Slower — write ingestion and query code |
| **Chunking control** | Fixed strategies only | 10+ chunking strategies including hierarchical |
| **Index types** | Vector store only | Vector, summary, knowledge graph, SQL |
| **Query routing** | Not available | RouterQueryEngine — auto-selects best index |
| **Multi-index** | Not available | Combine multiple indexes in one query |
| **Retrieval quality** | Good for standard use cases | Better for specialized domains |
| **Custom embedding** | Bedrock models only (Titan, Cohere) | Any HuggingFace model including fine-tuned |
| **Cost** | Bedrock pricing per token + OpenSearch | EC2/Lambda + OpenSearch (often cheaper at scale) |
| **Best for** | Standard RAG on PDFs, quick setup needed | Large doc sets, poor retrieval quality, custom chunking |
| **❌ Limitation** | Cannot customize chunking or query logic | More engineering effort |

### Use Case 3: LLM Observability and Debugging
| Dimension | CloudWatch + X-Ray + CloudTrail (AWS Native) | LangSmith (OSS Platform) |
|---|---|---|
| **What it is** | AWS monitoring services | Purpose-built LLM tracing platform |
| **LLM trace detail** | Must log manually in code | Automatic — full prompt, response, chunks |
| **Retrieved chunk visibility** | Manual logging required | Built-in, shows similarity scores |
| **Prompt comparison** | Build custom dashboards | Side-by-side experiment comparison UI |
| **Evaluation** | Build from scratch | Built-in eval framework with ground truth datasets |
| **Data location** | Stays in your AWS account | Sent to LangSmith cloud (self-hosted option exists) |
| **Regulated industries** | YES — safe by default | Self-hosted version required |
| **Cost** | CloudWatch pricing | Free tier; paid for production volume |
| **Best for** | Production compliance, regulated data | Development, debugging, RAG quality testing |
| **❌ Limitation** | Generic — not built for LLM debugging | Data leaves your AWS account (unless self-hosted) |

### Use Case 4: Data Pipeline Orchestration
| Dimension | AWS Step Functions (Native) | Apache Airflow on MWAA (OSS on AWS) |
|---|---|---|
| **What it is** | Serverless state machine service | Open-source workflow orchestrator |
| **Workflow definition** | JSON (Amazon States Language) | Python code |
| **Trigger model** | Event-driven (EventBridge, API, Lambda) | Schedule-based + event-based |
| **Complex DAGs** | Limited — designed for simple state machines | Excellent — complex dependency chains |
| **Visual UI** | Yes — built-in execution graph | Yes — Airflow UI with Gantt charts |
| **Retry logic** | Built-in per state | Built-in per task with configurable backoff |
| **Data engineering** | Not ideal | Industry standard — 500+ operators for all tools |
| **Serverless** | Yes — zero servers to manage | No — MWAA runs workers (you pay per hour) |
| **Cost** | Per state transition ($0.025 per 1,000) | MWAA worker cost (hourly) |
| **Best for** | Event-driven workflows, microservice orchestration | Data engineering teams, complex ETL pipelines |
| **❌ Limitation** | No Python — steep learning curve for complex DAGs | Higher cost, not serverless |

### Use Case 5: Data Transformation
| Dimension | AWS Glue ETL (Native) | dbt (data build tool — OSS) |
|---|---|---|
| **What it is** | Serverless Spark-based ETL service | SQL-first transformation framework |
| **Language** | Python or Scala (Spark) | SQL (any dialect: Redshift, Athena, Snowflake) |
| **Non-Spark transforms** | Not efficient | No Spark — pure SQL |
| **Version control** | Glue Studio or manual Git | Native Git — .sql files are code |
| **Testing** | Manual | Built-in: not_null, unique, accepted_values tests |
| **Documentation** | Manual | Auto-generated from schema + descriptions |
| **Data lineage** | Limited | Full visual lineage graph auto-generated |
| **Who writes it** | Data engineers (Python/Spark) | Analysts AND engineers (SQL) |
| **Cost** | Per DPU (Data Processing Unit) hour | Free (OSS core) — only pay for warehouse compute |
| **Best for** | Large Spark jobs, non-SQL transforms, streaming | SQL-based transforms, analytics teams, BI |
| **❌ Limitation** | Not SQL-friendly; harder to test | Cannot do Spark-level complex transforms |

### Use Case 6: ML (Machine Learning) Experiment Tracking
| Dimension | SageMaker Experiments (Native) | MLflow (OSS on AWS EC2/RDS) |
|---|---|---|
| **What it is** | AWS-managed experiment tracking | Open-source experiment tracking platform |
| **Framework support** | Best with SageMaker SDK | Any framework: sklearn, PyTorch, TensorFlow, XGBoost |
| **Portability** | AWS only | Runs on any cloud, on-prem, locally |
| **Model registry** | SageMaker Model Registry | MLflow Model Registry |
| **Model serving** | SageMaker Endpoints (managed) | MLflow Serving (you manage) or SageMaker |
| **UI quality** | Good, integrated with SageMaker Studio | Good open-source UI |
| **Setup effort** | Zero — part of SageMaker | Need to deploy EC2 + RDS + S3 |
| **Cost** | Included in SageMaker Studio domain | EC2 + RDS cost (~$50-100/month small setup) |
| **Best for** | Pure SageMaker pipelines, AWS-only | Multi-cloud, framework-agnostic, portability |
| **❌ Limitation** | Vendor lock-in — not portable | Self-managed infrastructure |

### Use Case 7: Text Embeddings
| Dimension | Bedrock Titan / Cohere Embed (Native) | HuggingFace sentence-transformers (OSS) |
|---|---|---|
| **What it is** | Managed embedding API — pay per token | Open-source models you download and run |
| **Setup** | One API call | Download model (~0.5-2 GB), run on EC2 GPU |
| **Cost model** | Per token (~$0.00002/1K tokens Titan) | EC2 GPU compute (~$0.53/hr for g4dn.xlarge) |
| **Cost at scale** | Gets expensive: 1B tokens = $20 | 1B tokens on GPU ≈ $5-10 in compute time |
| **Domain fine-tuning** | Not possible | YES — fine-tune on your insurance/banking corpus |
| **Data privacy** | Data sent to Bedrock endpoint | Data never leaves your EC2 instance |
| **Model quality** | Good general-purpose | Can be better for specialized domains if fine-tuned |
| **Dimensions** | Titan: 1536, Cohere v3: 1024 | BGE-large: 1024, BGE-m3: 1024 (multilingual) |
| **Best for** | Quick start, general English, low volume | High volume, domain-specific, privacy-critical |
| **❌ Limitation** | Cannot fine-tune; costs scale with volume | GPU infra to manage; slower to start |

---

## 🏗️ POC vs ENTERPRISE — Realistic Stack Choices

> **Mentor Note:** One of the most common interview mistakes is describing only one stack.
> A senior architect must know what to build for a POC (Proof of Concept) demo in week 1
> versus what a production enterprise system looks like in month 6. They are very different.

### POC Stack — "Prove it works in 1-2 weeks, minimal infrastructure"
```
Goal: Show a working demo to stakeholders. Speed > perfection.
Budget: ~$50-300/month

Compute:
  Single EC2 t3.medium or g4dn.xlarge (if GPU needed for local models)
  Docker Compose to run multiple services on one instance

AI Framework:
  LangChain → fastest RAG prototype
  Bedrock (Claude 3.5 Haiku) → cheapest capable model for demos
  Bedrock Titan Embeddings → no GPU needed, quick setup

Vector Store:
  ChromaDB (OSS, runs in-process, no server) → zero setup
  Or OpenSearch Serverless (1 OCU) → if you want a closer-to-prod setup

Storage:
  Single S3 bucket (no Bronze/Silver/Gold zones yet)

Orchestration:
  None — or simple Lambda with EventBridge schedule

Observability:
  LangSmith (free tier) → trace every RAG call during dev and demo

Data:
  Hand-picked 20-50 sample documents
  No data migration yet — just load from laptop to S3

Governance:
  None yet — not needed for internal demo

Total setup time: 2-3 days for a working demo
```

### Enterprise Production Stack — "Scale, govern, operate, audit"
```
Goal: Production system serving real users with SLA, compliance, and monitoring.
Budget: $5,000-50,000+/month depending on scale

Data Migration Layer:
  AWS Direct Connect (10 Gbps) — dedicated network
  AWS DMS (Database Migration Service) + CDC for databases
  AWS DataSync for file systems
  Snowball Edge for initial bulk load > 50 TB

Data Lake:
  S3 Bronze / Silver / Gold zones with versioning
  Glue Catalog + Lake Formation (fine-grained access control)
  Macie (auto-detect PII in S3)
  dbt (SQL transformations: Bronze → Silver → Gold)

AI Pipeline:
  LlamaIndex (ingestion) + HuggingFace fine-tuned embeddings (domain-specific)
  OpenSearch Serverless (vector store with k-NN)
  Bedrock (Claude 3.5 Sonnet) for complex queries
  HuggingFace fine-tuned model on SageMaker Endpoint for classification

Agent Layer:
  LangGraph on ECS Fargate (complex multi-step agent logic)
  DynamoDB (checkpointing, session state, conversation history)
  Step Functions (human-in-the-loop approval workflows)
  SQS FIFO (human review queues)

Observability:
  LangSmith (self-hosted on EC2 for regulated data) OR CloudWatch + custom dashboards
  X-Ray (distributed tracing)
  CloudTrail (all Bedrock API audit logs)

ML (Machine Learning) Operations:
  MLflow on EC2 + RDS + S3 (experiment tracking, model registry)
  SageMaker Model Monitor (data drift alerts)
  SageMaker Clarify (bias detection, explainability)
  Airflow on MWAA (daily/weekly pipeline scheduling)

Governance:
  Bedrock Guardrails (PII redaction, content filtering, grounding checks)
  Lake Formation (column/row-level data access control)
  AWS Config + Security Hub (continuous compliance)
  KMS CMK (Customer Master Key) encryption everywhere
  CloudHSM for FIPS 140-2 Level 3 (banking/OSFI compliance)

Total setup time: 3-6 months for full production deployment
```

### How to Talk About This in an Interview
```
Interviewer: "How would you architect a RAG system for our insurance company?"

Wrong answer (junior): "I would use Bedrock Knowledge Bases and Lambda."

Right answer (senior):
  "It depends on where we are in the lifecycle.

  For a POC to validate the concept with your team in two weeks:
  I would use LangChain with Bedrock (Claude Haiku) and OpenSearch Serverless.
  Fast setup, low cost, shows the value quickly.

  For production with the compliance requirements of an insurance company:
  I would separate the OSS orchestration layer from the AWS infrastructure layer.
  LangGraph handles the agent logic — we own the code, it is testable, auditable.
  LlamaIndex handles ingestion with hierarchical chunking tuned for policy documents.
  Fine-tuned HuggingFace sentence-transformers give us better retrieval quality
  on insurance-specific terminology than generic Bedrock Titan embeddings.
  All of this runs on AWS: ECS Fargate for the agent, OpenSearch Serverless for
  vectors, Aurora for structured data, DynamoDB for state, Bedrock Guardrails for
  PII (Personally Identifiable Information) redaction, and CloudTrail for audit.

  The OSS layer gives us control and portability.
  The AWS layer gives us security, scale, and compliance."
```

---

## 📚 UPDATED REFERENCE DOCS — OSS Tools + AWS

### OSS Framework Documentation
| Tool | Official Docs | Best Starting Point |
|---|---|---|
| LangChain | https://python.langchain.com/docs/introduction/ | RAG tutorial in docs |
| LangGraph | https://langchain-ai.github.io/langgraph/ | Quickstart: build your first agent |
| LangSmith | https://docs.smith.langchain.com/ | Tracing quickstart |
| LlamaIndex | https://docs.llamaindex.ai/en/stable/ | "Building RAG from Scratch" guide |
| MLflow | https://mlflow.org/docs/latest/index.html | Tracking quickstart |
| Apache Airflow | https://airflow.apache.org/docs/ | Tutorial: first DAG |
| dbt (data build tool) | https://docs.getdbt.com/ | Jaffle Shop tutorial (official beginner project) |
| Hugging Face Transformers | https://huggingface.co/docs/transformers/index | Pipeline quickstart |
| sentence-transformers | https://www.sbert.net/ | Semantic search tutorial |
| RAGAS (RAG evaluation) | https://docs.ragas.io/en/latest/ | Evaluate your first RAG pipeline |
| pgvector (PostgreSQL vector extension) | https://github.com/pgvector/pgvector | README — install and usage |

### AWS + OSS Integration
| Topic | Link |
|---|---|
| LangChain + AWS Bedrock | https://python.langchain.com/docs/integrations/providers/aws/ |
| LlamaIndex + AWS | https://docs.llamaindex.ai/en/stable/examples/llm/bedrock/ |
| MLflow on SageMaker | https://docs.aws.amazon.com/sagemaker/latest/dg/mlflow.html |
| Airflow on MWAA | https://docs.aws.amazon.com/mwaa/latest/userguide/what-is-mwaa.html |
| HuggingFace on SageMaker | https://huggingface.co/docs/sagemaker/index |
| dbt + Redshift | https://docs.getdbt.com/docs/core/connect-data-platform/redshift-setup |

### Learning Path — Recommended Order
```
Week 1: Foundation
  → Read LangChain RAG tutorial → build a simple PDF Q&A app
  → Set up LangSmith → trace your first RAG runs

Week 2: Data Layer
  → dbt Jaffle Shop tutorial → understand models, tests, lineage
  → AWS Glue getting started → run your first ETL job

Week 3: Advanced RAG
  → LlamaIndex quickstart → compare retrieval quality vs LangChain
  → Try hierarchical chunking on a 50-page PDF

Week 4: Agents
  → LangGraph quickstart → build your first stateful agent
  → Add human-in-the-loop with DynamoDB checkpointing

Week 5: Evaluation + Governance
  → RAGAS tutorial → evaluate your RAG pipeline with metrics
  → Bedrock Guardrails setup → add PII redaction to your pipeline

Week 6: Production Readiness
  → MLflow tracking tutorial → track model experiments
  → Airflow on MWAA → schedule your data pipeline
  → AWS Well-Architected ML Lens → review your architecture
```

---

*End of AI_DataMigration_RAG_AgenticAI.md — Built for 15-min interview excellence.*
