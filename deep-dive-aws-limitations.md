# Where AWS Bedrock Falls Short — Corrected Deep Dive

> **Note**: The previous version overstated several AWS limitations. This corrected version is based on verified documentation as of mid-2025. AWS has shipped capabilities that address many common criticisms.

---

## Corrections: What I Got Wrong

| Previous Claim | Reality (Fact-Checked) |
|---|---|
| ❌ "No built-in evaluation" | ✅ **Bedrock has RAG Evaluation (GA 2025)** — Faithfulness, Context Relevance, Context Coverage, Correctness, Completeness, Helpfulness, Logical Coherence. Supports custom metrics too. |
| ❌ "No per-user access control" | ✅ **Bedrock KB supports ACL-aware filtering** — syncs ACLs from SharePoint, Google Drive, Confluence, Box. Also supports metadata-based multi-tenancy in a single KB. |
| ❌ "Vector-only search, no hybrid" | ✅ **Hybrid search is supported** — for OpenSearch Serverless, Aurora PostgreSQL, and MongoDB Atlas vector stores (since April 2025). |
| ❌ "No reranking" | ✅ **Reranking is built-in** — service-managed reranking model by default, plus you can use custom reranking models via the Rerank API. |
| ❌ "Cannot handle infographics/charts" | ✅ **Multimodal parsing (GA Dec 2024)** — Bedrock Data Automation (BDA) and FM-based parsers extract figures, charts, tables, images. Nova Multimodal Embeddings for visual similarity search. |
| ❌ "No custom chunking" | ✅ **Custom transformation Lambda** — full control over chunking logic via your own Lambda function. |
| ❌ "Fixed RAG pipeline" | ⚠️ Partially true — but query decomposition, metadata filtering, reranking, hybrid search are all tunable knobs. |

---

## Slide 1: Customization — Revised Assessment

### What AWS Bedrock Actually Offers (Tunable Knobs)

| Customization Area | Available in Bedrock KB | Limitation Remains? |
|---|---|---|
| Chunking strategy | ✅ Fixed, Semantic, Hierarchical, None, **Custom Lambda** | No — fully customizable |
| Hybrid search (BM25 + vector) | ✅ Supported for OpenSearch, Aurora PG, MongoDB | No |
| Reranking | ✅ Built-in service-managed + custom reranker API | No |
| Query decomposition | ✅ Automatic multi-step query breakdown | No |
| Metadata filtering at query time | ✅ Complex filter expressions supported | No |
| Custom transformation (parsing) | ✅ Lambda-based custom transformation | No |
| Multimodal parsing (charts/tables) | ✅ BDA + FM parsers extract visual content | No |
| Custom prompt template | ⚠️ Some templating — less flexible than full code | **Partial limitation** |
| Multi-hop retrieval / agentic RAG | ✅ Via AgentCore with tool use | No |
| Custom scoring/retrieval algorithm | ❌ Cannot replace the core retrieval engine | **Real limitation** |
| Bring your own embedding model | ❌ Must use Bedrock-supported embeddings | **Real limitation** |

### What's ACTUALLY Limited

The real limitations that remain:

1. **Cannot bring arbitrary embedding models** — You're restricted to Titan Embed V2 and Cohere Embed. If you've trained a custom embedding model (e.g., fine-tuned E5 or BGE on your domain), you cannot use it with Bedrock KB.

2. **Cannot replace the retrieval algorithm** — The core retrieve → rank pipeline is fixed. You can tune parameters (top-K, reranking, hybrid weights), but you can't inject a completely custom retrieval strategy (e.g., ColBERT, custom graph-based retrieval).

3. **Limited prompt engineering in KB mode** — When using the managed RetrieveAndGenerate API, you have less control over the exact prompt template compared to building your own chain.

4. **No custom post-processing of retrieved chunks** — You can't insert logic between "chunks retrieved" and "sent to LLM" (e.g., summarize retrieved chunks before sending to model).

### Impact Assessment for 20 Complex PDFs

| Limitation | Impact | Workaround |
|---|---|---|
| Can't use custom embeddings | 🟡 Medium — generic embeddings work for most cases, but domain-specific jargon retrieval suffers | None within Bedrock KB |
| Fixed retrieval algorithm | 🟡 Medium — the built-in pipeline with hybrid + reranking is good for 80% of cases | Use AgentCore with custom tools for complex retrieval |
| Limited prompt control | 🟢 Low — you can work around this by using Retrieve API + custom generation | Use Retrieve API only, build generation yourself |

---

## Slide 2: Fine-tuning — Revised Assessment

### What Bedrock Actually Supports

| Capability | AWS Bedrock | Notes |
|---|---|---|
| Fine-tune LLM | ✅ Supported for Llama, Titan, Cohere, Mistral | **Claude is NOT fine-tunable** |
| Continued pre-training | ✅ Available for select models | Inject domain knowledge |
| Custom model import | ✅ Import fine-tuned models from anywhere | Deploy your own model on Bedrock |
| Distillation | ✅ Model distillation (large → small) | New capability |

### What Bedrock Does NOT Support

| Capability | Gap | Why It Matters |
|---|---|---|
| **Fine-tune embeddings** | ❌ Not supported | Cannot optimize retrieval for your domain vocabulary |
| **RLHF / DPO** | ❌ Not available in Bedrock | Cannot align model to your quality preferences via human feedback |
| **Fine-tune Claude** | ❌ Anthropic doesn't offer this via Bedrock | If Claude is your best model, you can't fine-tune it |
| **Online learning** | ❌ No continuous improvement from user interactions | Model stays static until manual retrain |
| **Fine-tune reranker** | ❌ Service-managed reranker is fixed | Cannot optimize ranking for your specific query patterns |

### Realistic Impact for 20-PDF KB

For a 20-PDF knowledge base, the fine-tuning gap matters ONLY IF:
- Your documents use highly specialized terminology that confuses generic embeddings
- You need the model to respond in a very specific format/style consistently
- You want the system to improve over time from user feedback

For **general business documents** (HR policies, technical guides, financial reports with standard terminology), fine-tuning is often unnecessary — the base models + good chunking + reranking handle it well.

**Honest assessment**: Fine-tuning matters more for Databricks' use case when you have **thousands of documents with domain-specific language** and need the embedding space to be optimized. For 20 PDFs, it's rarely the bottleneck.

---

## Slide 3: Evaluation — Revised Assessment

### AWS Bedrock Evaluation — What Actually Exists (GA 2025)

Bedrock now has **purpose-built RAG evaluation** with these built-in metrics:

| Metric | What It Measures | Type |
|---|---|---|
| **Context Relevance** | Are retrieved chunks relevant to the query? | Retrieval |
| **Context Coverage** | Do retrieved chunks cover all ground truth info? | Retrieval |
| **Correctness** | Is the answer factually correct? | Generation |
| **Completeness** | Does the answer address all aspects? | Generation |
| **Faithfulness** | Does the answer avoid hallucination vs context? | Generation |
| **Helpfulness** | Is the answer useful overall? | Generation |
| **Logical Coherence** | Is the answer internally consistent? | Generation |
| **Custom Metrics** | Define your own evaluation criteria | Both |

### Evaluation Comparison — Corrected

| Capability | AWS Bedrock Evaluations | Databricks (MLflow + Agent Eval) |
|---|---|---|
| Built-in RAG metrics | ✅ 7 built-in + custom | ✅ Via MLflow + RAGAS |
| LLM-as-a-judge | ✅ Built-in | ✅ Built-in |
| Custom metrics | ✅ Define custom prompts | ✅ Custom Python evaluators |
| Evaluate external RAG systems | ✅ Can evaluate non-Bedrock RAG | ✅ Any RAG system |
| Console UI for evaluation | ✅ Bedrock console | ✅ MLflow UI |
| Programmatic evaluation | ✅ API-based | ✅ API-based |
| **A/B comparison of configs** | ⚠️ Run separate jobs, compare manually | ✅ MLflow experiment comparison (side-by-side) |
| **CI/CD integration** | ⚠️ Possible but not built-in | ✅ Native MLflow CI/CD hooks |
| **Regression testing** | ⚠️ Must build test suite externally | ✅ Native experiment tracking with baselines |
| **Per-query cost tracking** | ❌ Aggregate CloudWatch only | ✅ Per-inference cost attribution |
| **Feedback loop (user → retrain)** | ❌ No built-in mechanism | ✅ MLflow + reinforcement pipeline |

### Remaining Evaluation Gaps (Honest)

The real remaining gaps in AWS Bedrock evaluation:

1. **No native A/B testing framework** — You can run two evaluation jobs with different configs, but there's no built-in "compare these two approaches side-by-side" UI like MLflow provides.

2. **No CI/CD-native regression testing** — You can't easily say "run this eval on every PR and block merge if faithfulness drops below 0.8." Databricks + MLflow makes this natural.

3. **No feedback loop** — There's no built-in way to capture "user said this answer was wrong" → feed into improvement pipeline. You'd build this yourself.

4. **No per-query cost tracking** — CloudWatch gives you aggregate costs, not "this specific query cost $0.003 because it retrieved 5 chunks and used 2000 tokens."

These are **workflow gaps**, not capability gaps. Bedrock can measure quality; it just doesn't have the MLOps loop around it that Databricks/MLflow provides.

---

## Slide 4: Complex Document Handling — Revised

### AWS Bedrock Multimodal Parsing (GA December 2024)

**I was wrong** — Bedrock KB now handles complex documents significantly better than I stated:

| Content Type | AWS Bedrock (Current) | Method |
|---|---|---|
| Plain text | ✅ Works well | Standard parsing |
| Tables | ✅ Extracted and structured | BDA parser / FM parser |
| Charts/Graphs | ✅ Extracted as images + described | BDA + FM-based description |
| Infographics | ✅ Parsed via multimodal model | FM parser generates text descriptions |
| Scanned PDFs | ✅ OCR via Textract integration | Automatic |
| Images within PDFs | ✅ Stored and retrievable | Nova Multimodal Embeddings |
| Multi-column layouts | ⚠️ Improved but imperfect | BDA handles most cases |
| Complex nested tables | ⚠️ May lose some relationships | FM parser helps |

### Two Multimodal Approaches in Bedrock KB

```
Approach 1: Bedrock Data Automation (BDA)
┌─────────────────────────────────────────────────┐
│ • Converts charts/images → text descriptions    │
│ • Extracts tables → structured text             │
│ • Processes audio, video, images, PDFs          │
│ • Stored as searchable text chunks              │
│ • Works with standard text embeddings           │
└─────────────────────────────────────────────────┘

Approach 2: Nova Multimodal Embeddings
┌─────────────────────────────────────────────────┐
│ • Embeds images directly (visual similarity)    │
│ • Query with an image → find similar images     │
│ • Charts are searchable by visual content       │
│ • No text conversion needed                     │
└─────────────────────────────────────────────────┘
```

### Remaining Gaps for Complex Documents

| Scenario | AWS Bedrock | Databricks | Winner |
|---|---|---|---|
| "What number is in the Q3 chart?" | ✅ BDA extracts chart values | ✅ Custom vision pipeline | **Tie** (both can do it) |
| "Compare the trend in charts A vs B" | ⚠️ Depends on BDA quality | ✅ Custom multi-modal reasoning | Databricks (more control) |
| Highly custom document format | ⚠️ BDA is generic | ✅ Train custom parser | Databricks |
| Standard business PDFs (tables + text) | ✅ Works well out of the box | ✅ Works with setup effort | **AWS** (faster) |

---

## Slide 5: Access Control — Revised (Bedrock ACL vs Unity Catalog)

### Bedrock KB Access Control — What Actually Exists

AWS Bedrock KB now supports **ACL-aware filtering**:

| Feature | AWS Bedrock KB (Current) | Unity Catalog |
|---|---|---|
| Document-level ACL | ✅ Syncs ACLs from connectors (SharePoint, Google Drive, Confluence, Box) | ✅ Row-level security |
| Metadata-based multi-tenancy | ✅ Single KB with tenant filters | ✅ Column/row masking |
| User/group-based filtering | ✅ Pass user context → filter at query time | ✅ Native identity integration |
| Pre-retrieval filtering | ✅ ACLs applied before vector search | ✅ Query predicates |
| Custom ACL via metadata | ✅ Tag docs with allowed users/groups | ✅ Tags + policies |
| Cross-connector ACL | ✅ Per-connector ACL sync | ✅ Universal catalog |

### Where Unity Catalog Still Wins

| Capability | Bedrock KB ACL | Unity Catalog | Notes |
|---|---|---|---|
| **Column-level masking** | ❌ All-or-nothing per document | ✅ Mask specific fields | UC can hide salary in otherwise-accessible HR doc |
| **Data lineage (chunk-level)** | ⚠️ Source attribution only | ✅ Full lineage graph | UC traces data from source → transform → embedding → query |
| **Centralized governance across ALL data** | ❌ Only KB scope | ✅ Governs tables, models, notebooks, ML | UC is the single governance layer |
| **Dynamic policies** | ⚠️ Static ACL sync | ✅ Attribute-based access control (ABAC) | UC policies can reference real-time attributes |
| **Audit across full ML lifecycle** | ⚠️ CloudTrail (API-level) | ✅ Query-level audit with data accessed | UC logs which data fed which answer |

### Honest Assessment

For the "20 PDFs, multi-team access" scenario:
- **If your connectors support ACL** (SharePoint, Confluence, Google Drive): Bedrock handles per-user filtering natively. No need for multiple KBs.
- **If you need column-level masking** within a document: Unity Catalog wins.
- **If you need centralized governance across data + ML + analytics**: Unity Catalog wins (it's an org-wide governance layer, not just for KB).

---

## Revised Summary: Honest Comparison

| Limitation | Severity (Corrected) | Was I Fair Before? |
|---|---|---|
| Custom embedding models | 🟡 Medium | Yes — this is a real gap |
| Fine-tuning (no RLHF, no Claude) | 🟡 Medium | Yes, but less relevant for 20 PDFs |
| Evaluation (no CI/CD loop, no A/B) | 🟡 Medium (workflow gap, not capability gap) | **No — I overstated this. Bedrock has solid eval metrics now.** |
| Complex doc handling | 🟢 Low (BDA + multimodal parsing exist) | **No — I was wrong. Bedrock handles charts/tables/images.** |
| Per-user access control | 🟢 Low (ACL-aware filtering exists) | **No — I was wrong. Bedrock has ACL sync.** |
| Customization (retrieval algorithm) | 🟡 Medium | Partially — hybrid + reranking exist, but core algo is fixed |
| Feedback loop / online learning | 🟡 Medium | Yes — this remains a gap |
| Multi-cloud | 🔴 Hard constraint | Yes — AWS only, no workaround |

### Bottom Line (Corrected)

> **AWS Bedrock is stronger than I initially presented.** It has evaluation, multimodal parsing, hybrid search, reranking, ACL filtering, and custom chunking. The "managed RAG" is not a toy — it's a production-capable system.
>
> **Databricks still wins on**: custom embedding models, RLHF, CI/CD-native evaluation workflows, centralized org-wide data governance (Unity Catalog scope), multi-cloud, and situations where you need the retrieval algorithm itself to be different.
>
> **For 20 complex PDFs**: AWS Bedrock (with BDA multimodal parsing + ACL filtering + built-in evaluation) is likely **sufficient** and will be production-ready in hours vs. days for Databricks. Databricks is justified only if you need custom embeddings, RLHF, or your cluster already exists and you want one platform for everything.
