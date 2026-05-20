# AI Project — On-Prem to AWS: Data Migration → RAG + Agentic AI + Governance
> **Interview Target:** CAD 140K+ | Senior Architect / AI Platform Engineer  
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

*End of AI_DataMigration_RAG_AgenticAI.md — Built for 15-min interview excellence.*
