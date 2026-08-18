# Where AWS Bedrock Falls Short — Deep Dive (Slide-Ready)

> Use each section below as a separate slide or slide pair in your presentation.

---

## Slide 1: Customization Limitations

### What You CAN'T Control in AWS Bedrock Knowledge Bases

| What You Want | AWS Bedrock Reality | Databricks Alternative |
|---|---|---|
| Custom retrieval logic | ❌ Fixed: embed → vector search → top-K | ✅ Any retrieval strategy (HyDE, reranking, multi-hop) |
| Query rewriting | ❌ No pre-processing of user query | ✅ Custom query transformation pipeline |
| Result re-ranking | ⚠️ Limited (only with Kendra reranker add-on) | ✅ Cross-encoder reranking, custom scoring |
| Hybrid search (BM25 + vector) | ❌ Vector-only in most stores | ✅ Full hybrid with tunable weights |
| Custom prompt template for RAG | ⚠️ Limited templating options | ✅ Complete prompt engineering control |
| Multi-index queries | ❌ One KB at a time (or merge awkwardly) | ✅ Query multiple indexes, fuse results |
| Chunk metadata filtering | ⚠️ Basic metadata filters only | ✅ Complex SQL-like filters on chunk metadata |
| Custom embedding dimensions | ❌ Locked to model's output dim | ✅ Dimensionality reduction, custom projections |

### Impact on Complex Document Q&A (20 PDFs with infographics)

```
User: "What's the ROI shown in the Q3 chart on page 47?"

AWS Bedrock:
┌─────────────────────────────────────────────┐
│ 1. Chunks the PDF → misses chart context    │
│ 2. Embeds text near chart → loses visual    │
│ 3. Retrieves text chunks → no chart data    │
│ 4. LLM hallucinates or says "not found"     │
│                                              │
│ Result: ❌ Cannot answer chart-based Qs     │
└─────────────────────────────────────────────┘

Databricks (with custom pipeline):
┌─────────────────────────────────────────────┐
│ 1. Custom parser extracts chart data        │
│    (using vision model or chart-to-table)   │
│ 2. Stores structured data alongside text    │
│ 3. Custom retrieval: text + structured      │
│ 4. LLM has actual chart values              │
│                                              │
│ Result: ✅ Accurate answer with numbers     │
└─────────────────────────────────────────────┘
```

### Workarounds (All Have Tradeoffs)

| Workaround | Complexity | Effectiveness |
|---|---|---|
| Pre-process PDFs before S3 upload (extract tables/charts externally) | High | Medium — loses context |
| Use Custom Lambda chunking | Medium | Limited — still can't change retrieval |
| Build custom RAG on Bedrock (skip KB, use raw APIs) | High | Good — but then why use Bedrock KB? |
| Use AgentCore with tool-use for structured data | Medium | Partial — complex to set up |

---

## Slide 2: Fine-tuning Gaps

### What Fine-tuning Means for a KB Q&A System

| Capability | Why It Matters | AWS Bedrock | Databricks |
|---|---|---|---|
| **Fine-tune the LLM** | Domain-specific language, internal jargon | ⚠️ Limited models only (Llama, Titan) | ✅ Any model (full fine-tune or LoRA) |
| **Fine-tune embeddings** | Better retrieval for YOUR documents | ❌ Not supported | ✅ Train domain-specific embeddings |
| **RLHF / DPO** | Align answers to your quality bar | ❌ Not available | ✅ Full RLHF/DPO pipeline |
| **Instruction tuning** | Teach specific response format | ⚠️ Basic via Bedrock fine-tuning | ✅ Complete control |
| **Continued pre-training** | Inject domain knowledge into model weights | ⚠️ Expensive, limited models | ✅ Via Mosaic AI training |

### Why Embedding Fine-tuning Matters (For 20 Complex PDFs)

```
Scenario: Your PDFs use internal terminology
- "EBITDA bridge" = specific internal report format
- "T2 metrics" = internal performance framework
- "Green Zone" = your company's compliance status

Generic embeddings (AWS Bedrock):
┌─────────────────────────────────────────────┐
│ "EBITDA bridge" → embeds near financial     │
│   generic content, not YOUR specific format │
│                                              │
│ Query: "Show me the EBITDA bridge for Q3"   │
│ Retrieved: random financial paragraphs      │
│ Quality: Poor 📉                            │
└─────────────────────────────────────────────┘

Fine-tuned embeddings (Databricks):
┌─────────────────────────────────────────────┐
│ "EBITDA bridge" → trained to be near YOUR   │
│   specific bridge format in YOUR docs       │
│                                              │
│ Query: "Show me the EBITDA bridge for Q3"   │
│ Retrieved: exact Q3 bridge section          │
│ Quality: Excellent 📈                       │
└─────────────────────────────────────────────┘
```

### AWS Bedrock Fine-tuning Constraints

| Constraint | Detail |
|---|---|
| Supported models | Only Llama, Titan, Cohere (NOT Claude!) |
| Training data format | Must be JSONL, specific schema |
| No embedding fine-tuning | Titan Embed V2 is frozen — take it or leave it |
| No RLHF | Cannot learn from user feedback loops |
| No online learning | Can't improve over time without full retrain |
| Cost | $8-12/hr for fine-tuning instances |
| No A/B testing of fine-tuned variants | Must deploy and manually compare |

---

## Slide 3: Evaluation Weaknesses

### What "Evaluation" Means for Production RAG

| Evaluation Type | What It Tests | AWS Bedrock | Databricks |
|---|---|---|---|
| **Retrieval quality** | Are the right chunks retrieved? | ❌ No built-in | ✅ MLflow + custom metrics |
| **Answer faithfulness** | Is the answer grounded in retrieved context? | ❌ No measurement | ✅ Automated faithfulness scoring |
| **Answer relevance** | Does it actually answer the question? | ⚠️ Basic (Bedrock Eval Jobs) | ✅ Custom relevance evaluators |
| **Regression testing** | Did the new chunking strategy break anything? | ❌ No test suites | ✅ MLflow experiment comparison |
| **A/B testing** | Which prompt/model/strategy is better? | ❌ Must build externally | ✅ Built into Agent Evaluation |
| **Human feedback loop** | Learn from user thumbs up/down | ❌ No built-in mechanism | ✅ Integrated with MLflow |
| **Cost tracking per query** | How much does each answer cost? | ⚠️ Aggregate only (CloudWatch) | ✅ Per-query cost attribution |

### The "Ship and Pray" Problem

```
AWS Bedrock workflow:
┌──────────────────────────────────────────────────────┐
│                                                       │
│  Build KB → Deploy → Users complain → ??? → Fix     │
│                                                       │
│  No systematic way to:                               │
│  • Measure answer quality before deploy              │
│  • Compare chunking strategies quantitatively        │
│  • Detect quality degradation over time              │
│  • Know WHICH documents cause bad answers            │
│                                                       │
└──────────────────────────────────────────────────────┘

Databricks workflow:
┌──────────────────────────────────────────────────────┐
│                                                       │
│  Build → Evaluate (automated) → Compare → Deploy    │
│     ↑                                    │           │
│     └──── Monitor + Feedback ←───────────┘           │
│                                                       │
│  Can measure:                                        │
│  • Retrieval precision/recall per question           │
│  • Faithfulness score per answer                     │
│  • Which docs/chunks perform worst                   │
│  • A/B test: "Did new chunking improve accuracy?"   │
│                                                       │
└──────────────────────────────────────────────────────┘
```

### Databricks Evaluation Example (MLflow)

```python
import mlflow
from databricks.agents import evaluate

# Define evaluation dataset
eval_data = [
    {"question": "What's the parental leave policy?",
     "expected_answer": "16 weeks paid...",
     "expected_source": "hr-policy-2024.pdf"},
    {"question": "What's the Q3 EBITDA?",
     "expected_answer": "$4.2M...",
     "expected_source": "q3-financials.pdf"},
]

# Run evaluation — measures retrieval + answer quality automatically
results = evaluate(
    model=rag_chain,
    data=eval_data,
    metrics=["faithfulness", "relevance", "retrieval_precision"],
)

# Compare two approaches
with mlflow.start_run(run_name="semantic-chunking"):
    results_v1 = evaluate(model=rag_v1, data=eval_data)

with mlflow.start_run(run_name="hierarchical-chunking"):
    results_v2 = evaluate(model=rag_v2, data=eval_data)

# MLflow UI shows: which approach wins per metric
```

### What This Means for Your 20 PDFs

With complex documents (hundreds of pages, infographics), you NEED evaluation to know:
- Is the chunking catching the right sections?
- Are chart-heavy pages being handled correctly?
- Does the model hallucinate when context is ambiguous?
- Which of the 20 PDFs causes the most wrong answers?

**AWS Bedrock cannot answer any of these questions systematically.**

---

## Slide 4: Complex Document Handling (Infographics & Charts)

### The 20-PDF Scenario: What Happens with Complex Content

| Content Type | AWS Bedrock | Databricks |
|---|---|---|
| Plain text paragraphs | ✅ Works well | ✅ Works well |
| Tables | ⚠️ Often loses structure | ✅ Custom table parser |
| Charts/Graphs | ❌ Treats as image → ignores | ✅ Vision model extraction |
| Infographics | ❌ Cannot extract meaning | ✅ Multi-modal pipeline |
| Scanned PDFs (OCR needed) | ⚠️ Basic Textract integration | ✅ Custom OCR + post-processing |
| Multi-column layouts | ⚠️ Sometimes jumbles columns | ✅ Layout-aware parsing |
| Headers/Footers/Page numbers | ⚠️ Included in chunks (noise) | ✅ Custom filtering |
| Cross-references ("see page 47") | ❌ No context linking | ✅ Custom reference resolution |

### Example: 200-Page PDF with Complex Infographics

```
Your PDF: "Annual Strategy Report 2024"
- 200 pages
- 30 charts/graphs
- 15 complex tables
- 20 infographics
- Mixed: text + visual + structured data

AWS Bedrock approach:
┌──────────────────────────────────────────┐
│ Step 1: Upload to S3                     │
│ Step 2: Bedrock KB ingests               │
│   - Extracts text (ignores images)       │
│   - Chunks at 300 tokens (or semantic)   │
│   - Many charts = empty/useless chunks   │
│ Step 3: User asks about chart data       │
│   - Retrieves text near chart            │
│   - No actual chart values available     │
│   - Answer: "I don't have that info"     │
│                                           │
│ Coverage: ~60% of document content        │
└──────────────────────────────────────────┘

Databricks approach:
┌──────────────────────────────────────────┐
│ Step 1: Upload to Unity Catalog Volume   │
│ Step 2: Custom pipeline:                 │
│   a. Layout detection (Unstructured.io)  │
│   b. Table extraction → structured data  │
│   c. Chart extraction → values via       │
│      vision model (GPT-4V/Claude Vision) │
│   d. Infographic → text description      │
│   e. Context-aware chunking              │
│ Step 3: User asks about chart data       │
│   - Retrieves structured chart data      │
│   - Answer includes actual numbers       │
│                                           │
│ Coverage: ~95% of document content        │
└──────────────────────────────────────────┘
```

---

## Summary: AWS Bedrock Limitation Severity

| Limitation | Severity for 20 Complex PDFs | Can You Work Around It? |
|---|---|---|
| **No custom retrieval logic** | 🔴 High — limits answer quality | Partially (custom app, skip KB) |
| **No embedding fine-tuning** | 🟡 Medium — hurts domain-specific queries | No workaround |
| **No systematic evaluation** | 🔴 High — can't measure or improve quality | Build externally (expensive) |
| **Poor infographic handling** | 🔴 High — 40% of content may be missed | Pre-process externally (complex) |
| **No per-user access control** | 🟡 Medium — depends on team structure | Multiple KBs (maintenance burden) |
| **No RLHF** | 🟡 Medium — can't learn from feedback | No workaround |
| **No A/B testing** | 🟡 Medium — can't compare approaches | Build custom metrics externally |

### Bottom Line

> AWS Bedrock is excellent for **"good enough, fast"** — text-heavy PDFs where 80% accuracy is acceptable and you need it live in minutes.
>
> Databricks is necessary when you need **"production-grade accuracy"** — complex documents, measurable quality, continuous improvement, and fine-grained access control.
