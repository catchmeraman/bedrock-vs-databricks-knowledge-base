# AWS Bedrock/AgentCore vs Databricks — Knowledge Base Q&A Architecture

Architecture comparison for hosting knowledge bases (PDFs, documents) and building Q&A systems.

## 📁 Files

- `bedrock-vs-databricks-kb-architecture.drawio` — Full architecture diagram (draw.io XML with AWS icons)

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
| **🆕 S3 Vectors** | ✅ **Lowest** (~$0.023/GB) | Medium | Cost-sensitive, low-traffic |
| **OpenSearch Serverless** | ~$350/mo min (2 OCUs) | Low | High-traffic production |
| **Aurora pgvector** | RDS instance pricing | Low | Already using Aurora |
| **Neptune Analytics** | Managed | Medium | Graph + vector hybrid |
| **Pinecone** | External SaaS | Low | Multi-cloud portability |
| **Redis Enterprise** | Cache pricing | Ultra-low | Real-time apps |
| **MongoDB Atlas** | External SaaS | Low | Document + vector |

### Databricks

| Vector Store | Cost | Notes |
|---|---|---|
| **Databricks Vector Search** | Cluster-based | Only managed option |
| FAISS / Chroma / Pinecone | External setup | Need separate infra |

---

## 💰 Cost Comparison (10,000 PDFs Knowledge Base)

| Component | AWS Bedrock + S3 Vectors | AWS Bedrock + OpenSearch | Databricks |
|---|---|---|---|
| **Vector Storage** | ✅ ~$0.023/GB (S3 pricing!) | ~$0.24/hr per OCU (min 2) | Cluster cost ($0.40-2.00/hr) |
| **Idle Cost** | ✅ **$0** (storage only) | ~$350/mo (min OCUs) | ❌ $500-2000/mo |
| **Query Cost** | Pay per query + tokens | Pay per query + tokens | DBU/hr + model serving |
| **Best For** | Low-traffic, cost-sensitive | High-traffic, low-latency | Full control, multi-cloud |

---

## ✅ Pros & Cons

| Criteria | AWS Bedrock / AgentCore | Databricks (Mosaic AI) |
|----------|------------------------|------------------------|
| **Setup Complexity** | ✅ Low — no code needed | ⚠️ Medium — notebooks required |
| **Time to Production** | ✅ Minutes | ⚠️ Hours/Days |
| **Chunking** | ✅ Auto (5 strategies) | ✅ Full custom control |
| **Models** | ✅ 30+ (Claude, Nova, Llama) | DBRX, Llama, Mixtral, External |
| **Agents** | ✅ AgentCore (tools, memory) | ✅ Mosaic AI Agents |
| **Guardrails** | ✅ Built-in (PII/topic/content) | ⚠️ Custom implementation |
| **Citations** | ✅ Built-in | ⚠️ Must implement |
| **Pricing** | ✅ Pay-per-query (no idle) | ❌ Cluster-based (idle cost) |
| **Customization** | ⚠️ Limited to config | ✅ Full code control |
| **Fine-tuning** | Limited | ✅ Full fine-tuning + RLHF |
| **Evaluation** | Bedrock Eval Jobs | ✅ MLflow + Agent Eval |
| **Multi-Cloud** | ❌ AWS only | ✅ AWS, Azure, GCP |
| **Data Governance** | IAM + Lake Formation | ✅ Unity Catalog |

---

## 🎯 When to Choose

| Choose | When... |
|--------|---------|
| **AWS Bedrock/AgentCore** | Quick setup, fully managed, pay-per-use, built-in guardrails/citations, AWS-native, cost-sensitive (S3 Vectors!) |
| **Databricks** | Full code control, custom RAG pipelines, fine-tuning needed, multi-cloud requirement, existing Databricks Lakehouse |

---

## 🔗 Resources

- [AWS Bedrock Knowledge Bases](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html)
- [Bedrock AgentCore](https://docs.aws.amazon.com/bedrock-agentcore/)
- [S3 Vectors (New!)](https://aws.amazon.com/s3/features/vectors/)
- [Databricks Mosaic AI](https://docs.databricks.com/en/generative-ai/agent-framework/index.html)
- [Databricks Vector Search](https://docs.databricks.com/en/generative-ai/vector-search.html)
