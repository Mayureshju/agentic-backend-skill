# Observability & Evaluation Reference

## Table of Contents
1. [Observability Architecture](#observability-architecture)
2. [Langfuse Integration](#langfuse-integration)
3. [LangSmith Integration](#langsmith-integration)
4. [Arize Phoenix Integration](#arize-phoenix-integration)
5. [RAG Evaluation (RAGAS)](#rag-evaluation-ragas)
6. [Cost Tracking](#cost-tracking)
7. [Production Monitoring](#production-monitoring)

---

## Observability Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    APPLICATION LAYER                         │
│                                                             │
│  API Request → Auth → Pipeline → LLM Call → Response        │
│       │          │        │          │          │            │
│       ▼          ▼        ▼          ▼          ▼            │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              TRACING / INSTRUMENTATION               │    │
│  │  • Request traces with span hierarchy               │    │
│  │  • Token usage per LLM call                         │    │
│  │  • Latency per pipeline stage                       │    │
│  │  • Retrieval quality scores                         │    │
│  └───────────────────┬─────────────────────────────────┘    │
└──────────────────────┼──────────────────────────────────────┘
                       ▼
┌──────────────────────────────────────────────────────────────┐
│              OBSERVABILITY BACKEND                            │
│  ┌───────────┐  ┌───────────┐  ┌──────────────────────────┐ │
│  │  Langfuse  │  │ LangSmith │  │    Arize Phoenix         │ │
│  │  (or)      │  │  (or)     │  │    (or)                  │ │
│  └───────────┘  └───────────┘  └──────────────────────────┘ │
│                                                              │
│  • Trace visualization        • Prompt management            │
│  • Cost dashboards            • Evaluation pipelines         │
│  • Quality metrics            • Alerting on regressions      │
└──────────────────────────────────────────────────────────────┘
```

### What to Trace

Every AI pipeline request should produce a trace with these spans:

| Span | What to Capture |
|------|----------------|
| `request` | Full request lifecycle, user ID, auth method |
| `embedding` | Model used, input length, latency, token count |
| `retrieval` | Strategy (vector/hybrid), num results, top similarity score |
| `reranking` | Model used, input/output count, latency |
| `generation` | Model, prompt tokens, completion tokens, latency, finish reason |
| `total` | End-to-end latency, total cost |

---

## Langfuse Integration

Langfuse is the recommended open-source option. Self-hostable, works with all frameworks.

### Dependencies

```
# Python
langfuse>=2.56.0

# JavaScript
langfuse: ^3.32.0
```

### Python — Direct Integration

```python
from langfuse import Langfuse
from langfuse.decorators import observe, langfuse_context
from app.config import settings

# Initialize client
langfuse = Langfuse(
    public_key=settings.langfuse_public_key,
    secret_key=settings.langfuse_secret_key,
    host=settings.langfuse_host,  # "https://cloud.langfuse.com" or self-hosted URL
)

# Decorate pipeline functions with @observe
@observe()
async def rag_query(question: str, user_id: str) -> dict:
    langfuse_context.update_current_trace(
        user_id=user_id,
        metadata={"source": "api"},
    )

    # 1. Embed
    query_embedding = await embed_query(question)

    # 2. Retrieve
    chunks = await retrieve(question, query_embedding)

    # 3. Generate
    response = await generate(question, chunks)

    # Score the trace (can also be done async later)
    langfuse_context.score_current_trace(
        name="user_feedback",
        value=1,  # Will be updated by user feedback endpoint
    )

    return response

@observe(as_type="generation")
async def generate(question: str, context_chunks: list) -> dict:
    """Traced as an LLM generation span."""
    # LLM call here — Langfuse auto-captures model, tokens, latency
    response = await llm.chat(messages)
    return response

@observe()
async def embed_query(text: str) -> list[float]:
    """Traced as a span under the parent trace."""
    return await embedder.embed(text)

@observe()
async def retrieve(query: str, embedding: list[float]) -> list[dict]:
    """Traced as a span capturing retrieval metrics."""
    results = await searcher.hybrid_search(query, embedding)
    langfuse_context.update_current_observation(
        metadata={"num_results": len(results), "strategy": "hybrid"},
    )
    return results
```

### Python — LangChain + Langfuse

```python
from langfuse.callback import CallbackHandler

langfuse_handler = CallbackHandler(
    public_key=settings.langfuse_public_key,
    secret_key=settings.langfuse_secret_key,
    host=settings.langfuse_host,
)

# Pass to any LangChain/LangGraph invocation
result = await chain.ainvoke(
    {"question": query},
    config={"callbacks": [langfuse_handler]},
)
```

### Python — LlamaIndex + Langfuse

```python
from langfuse.llama_index import LlamaIndexInstrumentor

instrumentor = LlamaIndexInstrumentor()
instrumentor.start()

# All LlamaIndex calls are now automatically traced
response = await query_engine.aquery("What is our refund policy?")

instrumentor.flush()
```

### TypeScript — Langfuse

```typescript
import { Langfuse } from "langfuse";
import { observeOpenAI } from "langfuse";

const langfuse = new Langfuse({
  publicKey: config.langfusePublicKey,
  secretKey: config.langfuseSecretKey,
  baseUrl: config.langfuseHost,
});

// Wrap OpenAI client for automatic tracing
const openai = observeOpenAI(new OpenAI(), { client: langfuse });

// Manual tracing for pipeline stages
async function ragQuery(question: string, userId: string) {
  const trace = langfuse.trace({ name: "rag-query", userId });

  const embeddingSpan = trace.span({ name: "embedding" });
  const queryEmbedding = await embedder.embed(question);
  embeddingSpan.end();

  const retrievalSpan = trace.span({ name: "retrieval" });
  const chunks = await searcher.hybridSearch(question, queryEmbedding);
  retrievalSpan.end({ metadata: { numResults: chunks.length } });

  const generation = trace.generation({
    name: "llm-generation",
    model: "gpt-4o",
    input: [{ role: "user", content: question }],
  });
  const response = await openai.chat.completions.create({ /* ... */ });
  generation.end({
    output: response.choices[0].message.content,
    usage: { promptTokens: response.usage?.prompt_tokens, completionTokens: response.usage?.completion_tokens },
  });

  await langfuse.flushAsync();
  return response;
}
```

---

## LangSmith Integration

Best when using LangChain/LangGraph — automatic trace capture with minimal setup.

### Dependencies
```
# Python
langsmith>=0.2.0
# Set environment variables:
# LANGCHAIN_TRACING_V2=true
# LANGCHAIN_API_KEY=your-langsmith-api-key
# LANGCHAIN_PROJECT=your-project-name

# JavaScript
# @langchain/core already includes LangSmith integration
# Set the same environment variables
```

### Python — Auto-Tracing

```python
import os
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = settings.langsmith_api_key
os.environ["LANGCHAIN_PROJECT"] = settings.app_name

# That's it — all LangChain/LangGraph calls are automatically traced
# No code changes needed in the pipeline

# For custom spans outside LangChain:
from langsmith import traceable

@traceable(name="custom-retrieval")
async def retrieve_documents(query: str) -> list[dict]:
    # Your retrieval logic
    pass
```

### TypeScript — Auto-Tracing

```typescript
// Set environment variables
process.env.LANGCHAIN_TRACING_V2 = "true";
process.env.LANGCHAIN_API_KEY = config.langsmithApiKey;
process.env.LANGCHAIN_PROJECT = config.appName;

// All @langchain/* calls are automatically traced
```

---

## Arize Phoenix Integration

Framework-agnostic, built on OpenTelemetry. Best for self-hosted observability.

### Dependencies
```
# Python
arize-phoenix>=7.0.0
openinference-instrumentation-openai>=0.1.0   # For OpenAI
openinference-instrumentation-langchain>=0.1.0 # For LangChain
openinference-instrumentation-llama-index>=3.0.0 # For LlamaIndex

# JavaScript
# Use OpenTelemetry directly with Phoenix collector
```

### Python — Phoenix Setup

```python
import phoenix as px
from openinference.instrumentation.openai import OpenAIInstrumentor
# Or: from openinference.instrumentation.langchain import LangChainInstrumentor

# Launch Phoenix (local development)
px.launch_app()

# For production, point to a remote Phoenix instance:
# os.environ["PHOENIX_COLLECTOR_ENDPOINT"] = "http://phoenix:6006"

# Instrument your LLM client
OpenAIInstrumentor().instrument()
# Or: LangChainInstrumentor().instrument()

# All OpenAI/LangChain calls are now automatically traced to Phoenix
```

---

## RAG Evaluation (RAGAS)

Evaluate RAG pipeline quality using standard metrics. Run evaluations on a schedule or triggered by user feedback.

### Dependencies
```
# Python
ragas>=0.2.0
datasets>=3.2.0
```

### Evaluation Metrics

| Metric | What it Measures | Range |
|--------|-----------------|-------|
| **Faithfulness** | Is the answer grounded in the retrieved context? No hallucination. | 0-1 |
| **Answer Relevancy** | Is the answer relevant to the question? | 0-1 |
| **Context Precision** | Are the retrieved chunks relevant to the question? | 0-1 |
| **Context Recall** | Does the retrieved context cover all parts of the ground truth? | 0-1 |

### Python — RAGAS Evaluation

```python
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy, context_precision, context_recall
from ragas import EvaluationDataset, SingleTurnSample

# Build evaluation dataset from traced data
samples = []
for trace in traces:
    samples.append(SingleTurnSample(
        user_input=trace["question"],
        response=trace["answer"],
        retrieved_contexts=[c["content"] for c in trace["chunks"]],
        reference=trace.get("ground_truth", ""),  # Optional
    ))

eval_dataset = EvaluationDataset(samples=samples)

# Run evaluation
result = evaluate(
    dataset=eval_dataset,
    metrics=[faithfulness, answer_relevancy, context_precision, context_recall],
)

print(result)
# {
#   "faithfulness": 0.87,
#   "answer_relevancy": 0.92,
#   "context_precision": 0.78,
#   "context_recall": 0.85,
# }
```

### Evaluation API Endpoint

```
POST /api/v1/evaluate                # Run RAGAS evaluation on recent traces
GET  /api/v1/evaluate/results        # Get evaluation results history
POST /api/v1/feedback                # Submit user feedback (thumbs up/down + optional text)
```

### Lightweight Evaluation (without RAGAS)

For simpler setups, implement LLM-as-a-Judge:

```python
async def evaluate_faithfulness(question: str, answer: str, context: str, llm) -> float:
    """Use LLM to judge if the answer is grounded in the context."""
    prompt = f"""Given the context and the answer, judge if the answer is faithful to the context.
Score from 0 to 1 where 1 means fully faithful (no hallucination).

Context: {context}
Answer: {answer}

Return only a number between 0 and 1."""

    response = await llm.chat([LLMMessage(role="user", content=prompt)])
    try:
        return float(response.content.strip())
    except ValueError:
        return 0.5
```

---

## Cost Tracking

### Token Usage Model

```python
@dataclass
class TokenUsage:
    prompt_tokens: int
    completion_tokens: int
    total_tokens: int
    embedding_tokens: int = 0
    model: str = ""
    estimated_cost_usd: float = 0.0

# Cost per 1M tokens (update these as pricing changes)
MODEL_PRICING = {
    "gpt-4o": {"input": 2.50, "output": 10.00},
    "gpt-4o-mini": {"input": 0.15, "output": 0.60},
    "gpt-4.1": {"input": 2.00, "output": 8.00},
    "gpt-4.1-mini": {"input": 0.40, "output": 1.60},
    "claude-sonnet-4-20250514": {"input": 3.00, "output": 15.00},
    "claude-haiku-4-5-20251001": {"input": 0.80, "output": 4.00},
    "text-embedding-3-small": {"input": 0.02, "output": 0.0},
    "text-embedding-3-large": {"input": 0.13, "output": 0.0},
}

def estimate_cost(usage: TokenUsage) -> float:
    pricing = MODEL_PRICING.get(usage.model, {"input": 0, "output": 0})
    cost = (usage.prompt_tokens * pricing["input"] + usage.completion_tokens * pricing["output"]) / 1_000_000
    if usage.embedding_tokens:
        embed_pricing = MODEL_PRICING.get("text-embedding-3-small", {"input": 0.02})
        cost += usage.embedding_tokens * embed_pricing["input"] / 1_000_000
    return round(cost, 6)
```

### Cost Tracking Middleware (Python)

```python
class CostTrackingMiddleware:
    """Accumulates token usage across a request and logs it."""

    async def __call__(self, request, call_next):
        request.state.token_usage = TokenUsage(0, 0, 0)
        response = await call_next(request)

        usage = request.state.token_usage
        if usage.total_tokens > 0:
            usage.estimated_cost_usd = estimate_cost(usage)
            logger.info(
                "request_cost",
                model=usage.model,
                prompt_tokens=usage.prompt_tokens,
                completion_tokens=usage.completion_tokens,
                embedding_tokens=usage.embedding_tokens,
                estimated_cost_usd=usage.estimated_cost_usd,
                request_id=request.state.request_id,
            )
        return response
```

---

## Production Monitoring

### Key Metrics to Track

| Category | Metric | Alert Threshold |
|----------|--------|----------------|
| **Latency** | P50/P95/P99 end-to-end | P95 > 5s |
| **Retrieval** | Avg chunks returned per query | < 1 chunk (empty retrieval) |
| **Retrieval** | Avg similarity score | < 0.3 (bad embeddings or queries) |
| **Generation** | Token usage per request | > 10k tokens (cost spike) |
| **Quality** | Faithfulness score (sampled) | < 0.7 |
| **Errors** | LLM API error rate | > 5% |
| **Cost** | Daily cost | > budget threshold |

### Health Check Extensions

Add pipeline-specific health checks:

```python
@health_router.get("/health")
async def health_check():
    checks = {}

    # Database
    checks["database"] = await check_db()

    # Embedding service
    try:
        test_embed = await embedder.embed("health check")
        checks["embedding"] = "healthy" if len(test_embed) > 0 else "unhealthy"
    except Exception as e:
        checks["embedding"] = f"unhealthy: {e}"

    # LLM provider
    try:
        test_response = await llm.chat([LLMMessage(role="user", content="ping")])
        checks["llm"] = "healthy" if test_response.content else "unhealthy"
    except Exception as e:
        checks["llm"] = f"unhealthy: {e}"

    # Observability backend
    if settings.langfuse_host:
        try:
            langfuse.auth_check()
            checks["observability"] = "healthy"
        except Exception:
            checks["observability"] = "unhealthy"

    is_healthy = all("healthy" == v for v in checks.values())
    return {"status": "healthy" if is_healthy else "degraded", "checks": checks}
```

### Environment Variables for Observability

```bash
# Langfuse
LANGFUSE_PUBLIC_KEY=pk-lf-...
LANGFUSE_SECRET_KEY=sk-lf-...
LANGFUSE_HOST=https://cloud.langfuse.com  # or self-hosted URL

# LangSmith
LANGCHAIN_TRACING_V2=true
LANGCHAIN_API_KEY=ls-...
LANGCHAIN_PROJECT=ai-backend

# Arize Phoenix
PHOENIX_COLLECTOR_ENDPOINT=http://localhost:6006

# Cost tracking
COST_TRACKING_ENABLED=true
COST_ALERT_DAILY_USD=50.00
```
