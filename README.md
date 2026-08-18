# AWS Bedrock/AgentCore vs Databricks — Knowledge Base Q&A Architecture

Architecture comparison for hosting knowledge bases (PDFs, documents) and building Q&A systems.

## 📁 Files

- `bedrock-vs-databricks-kb-architecture.drawio` — Full architecture diagram (draw.io XML with AWS icons)
- `deep-dive-aws-limitations.md` — Detailed analysis of where AWS Bedrock falls short

## 🏗️ Architecture Overview

The diagram compares both platforms side-by-side for a PDF-based Knowledge Base Q&A use case:

### AWS Bedrock / AgentCore (Managed RAG)
```
Data Connectors (10+) → Bedrock Knowledge Base → Auto Chunking → Embedding
    → Vector Store (7 options) → Foundation Model → Guardrails → Response with Citations
```

### Databricks (Mosaic AI)
```
Unity Catalog → Custom Chunking (Code) → Embedding → Vector Search
    → Model Serving → RAG Chain (LangChain) → Response (Custom)
```

---

## 📥 AWS Data Source Connectors (10+)

| Connector | Description |
|-----------|-------------|
| **Amazon S3** | PDFs, Docs, CSVs, images |
| **Web Crawler** | Public URLs / websites |
| **Confluence** | Atlassian wiki pages |
| **SharePoint** | Microsoft documents |
| **Salesforce** | CRM knowledge articles |
| **Custom (Lambda)** | Any source via code |
| **ServiceNow** | ITSM articles |
| **Jira** | Issue tracking |
| **GitHub** | Repos / wikis |
| **Google Drive** | Docs / sheets |
| **Microsoft Teams** | Messages / files |

> Databricks has NO native SaaS connectors — requires custom ingestion pipelines.

---

## 🗄️ Vector Store Options

### AWS Bedrock (7+ options)

| Vector Store | Cost | Latency | Best For |
|---|---|---|---|
| **🆕 S3 Vectors** | Lowest (~$0.023/GB) | Medium | Cost-sensitive, low-traffic |
| **OpenSearch Serverless** | ~$350/mo min (2 OCUs) | Low | High-traffic production |
| **Aurora pgvector** | RDS instance pricing | Low | Already using Aurora |
| **Neptune Analytics** | Managed | Medium | Graph + vector hybrid |
| **Pinecone** | External SaaS | Low | Multi-cloud portability |
| **Redis Enterprise** | Cache pricing | Ultra-low | Real-time apps |
| **MongoDB Atlas** | External SaaS | Low | Document + vector |

### Databricks

| Vector Store | Cost | Notes |
|---|---|---|
| **Databricks Vector Search** | Included in existing cluster | Only managed option |
| FAISS / Chroma / Pinecone | External setup | Need separate infra |

> **Note on cost**: If you already run a Databricks Lakehouse cluster for analytics/ETL, Vector Search runs on shared compute — no additional dedicated infrastructure needed. The "idle cost" concern is largely neutralized if the cluster is already justified for other workloads.

---

## 💰 Cost Comparison (Realistic: ~20 PDFs, some with 100s of pages + complex infographics)

| Component | AWS Bedrock + S3 Vectors | AWS Bedrock + OpenSearch | Databricks |
|---|---|---|---|
| **Vector Storage** | ~$0.023/GB | ~$350/mo min (2 OCUs) | Shared cluster (already running) |
| **Incremental Cost** | ✅ Near-zero (small dataset) | Disproportionate for 20 docs | ✅ Near-zero if cluster exists |
| **Query Cost** | Pay per query + tokens | Pay per query + tokens | Included in DBU |
| **Complex PDF parsing** | Basic (misses infographics) | Basic (misses infographics) | ✅ Custom parsers possible |
| **Best For** | Simple docs, quick start | N/A (overpriced for 20 PDFs) | Complex docs needing custom parsing |

### Key Insight: For ~20 PDFs
- **AWS Bedrock + S3 Vectors**: Cheapest if docs are text-heavy and straightforward
- **OpenSearch**: Overkill — minimum $350/mo for 20 documents is wasteful
- **Databricks**: If cluster already exists for Lakehouse, incremental cost ≈ $0. Better for complex documents that need custom extraction

---

## ✅ Updated Pros & Cons (Adjusted for Real-World Weighting)

| Criteria | Weight | AWS Bedrock / AgentCore | Databricks (Mosaic AI) |
|----------|--------|------------------------|------------------------|
| **Setup Speed** | High | ✅ Minutes, no code | ⚠️ Hours/Days |
| **Complex Doc Handling** | High | ⚠️ Limited (struggles with infographics) | ✅ Custom parsers |
| **Customization** | High | ❌ See detailed analysis below | ✅ Full control |
| **Fine-tuning** | High | ❌ See detailed analysis below | ✅ Full RLHF pipeline |
| **Evaluation** | High | ❌ See detailed analysis below | ✅ MLflow + Agent Eval |
| **Data Governance** | High | Basic (IAM + Lake Formation) | ✅ Unity Catalog (see below) |
| **Guardrails** | Medium | ✅ Built-in | ⚠️ Must implement |
| **Citations** | Medium | ✅ Built-in | ⚠️ Must implement |
| **Multi-Cloud** | Medium | ❌ AWS only | ✅ AWS, Azure, GCP |
| **Idle Cost** | Low | ✅ Pay-per-query | Shared cluster (not a factor) |
| **Models** | Medium | ✅ 30+ (Claude, Nova) | DBRX, Llama, External |
| **Agents** | Medium | ✅ AgentCore | ✅ Mosaic AI Agents |

---

## ⚠️ Where AWS Bedrock Falls Short (Deep Dive)

### See `deep-dive-aws-limitations.md` for full slide-ready content covering:

1. **Customization Limitations** — What you can't change, workarounds, impact
2. **Fine-tuning Gaps** — No RLHF, no embedding fine-tuning, model constraints
3. **Evaluation Weaknesses** — No systematic RAG evaluation, no regression testing
4. **Complex Document Handling** — Infographic/table extraction challenges

---

## 🛡️ Unity Catalog — Why It Matters for KB Governance

### What Unity Catalog Provides That AWS Doesn't

| Capability | Unity Catalog (Databricks) | AWS (IAM + Lake Formation) |
|---|---|---|
| **Row-level security** | ✅ Filter KB results by user's data access | ❌ Not supported in Bedrock KB |
| **Column-level masking** | ✅ Mask sensitive fields in responses | ❌ All-or-nothing access |
| **Data lineage** | ✅ Track: which PDF → which chunk → which answer | ⚠️ Basic (no chunk-level) |
| **Access audit** | ✅ Who queried what, which source was cited | ⚠️ CloudTrail (API-level only) |
| **Cross-workspace sharing** | ✅ Share KB across teams with fine-grained ACLs | ⚠️ Cross-account = complex |
| **Tagging & classification** | ✅ Tag documents as PII/Confidential, auto-filter | ❌ Manual S3 tagging only |
| **Dynamic access control** | ✅ "Sales team sees sales docs, HR sees HR docs" | ❌ Requires separate KBs per group |

### Real-World Example: Multi-Team KB

```
Scenario: 20 PDFs — some HR policies, some Engineering docs, some Finance reports

Unity Catalog approach:
┌─────────────────────────────────────────────────────┐
│ Single Vector Search Index (all 20 PDFs)            │
│                                                      │
│ User: "What's the parental leave policy?"           │
│                                                      │
│ UC Filter: user.team = "Engineering"                │
│ → Returns HR policy (public) ✅                      │
│ → Filters out Finance salary data ❌                 │
│ → Filters out Executive board notes ❌               │
└─────────────────────────────────────────────────────┘

AWS Bedrock approach:
┌─────────────────────────────────────────────────────┐
│ Option A: Single KB (no row-level filtering)        │
│ → Everyone sees everything 😬                        │
│                                                      │
│ Option B: Multiple KBs per team                     │
│ → HR KB, Engineering KB, Finance KB                 │
│ → Maintenance nightmare for 3+ teams                │
│ → Cross-team queries impossible                     │
└─────────────────────────────────────────────────────┘
```

---

## 🎯 When to Choose

| Choose | When... |
|--------|---------|
| **AWS Bedrock/AgentCore** | Simple text-heavy PDFs, quick MVP, small team, no fine-grained access control needed, want guardrails/citations out of the box |
| **Databricks** | Complex documents (infographics, tables), need fine-tuning, need per-user document access control (Unity Catalog), existing Lakehouse, multi-cloud, need systematic evaluation |

---

## 🔗 Resources

- [AWS Bedrock Knowledge Bases](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html)
- [Bedrock AgentCore](https://docs.aws.amazon.com/bedrock-agentcore/)
- [S3 Vectors](https://aws.amazon.com/s3/features/vectors/)
- [Databricks Mosaic AI](https://docs.databricks.com/en/generative-ai/agent-framework/index.html)
- [Databricks Vector Search](https://docs.databricks.com/en/generative-ai/vector-search.html)
- [Unity Catalog](https://docs.databricks.com/en/data-governance/unity-catalog/index.html)
