---
name: ai-backend-setup
description: >
  Set up a production-ready GenAI backend with interactive configuration. Use this skill whenever the user wants to
  create an AI backend, build a RAG system, build an agentic RAG application, create AI pipelines, set up an
  LLM-powered API, scaffold a GenAI project, create a vector search backend, build an enterprise AI service, set up
  document ingestion pipelines, or mentions anything related to setting up a new backend involving AI/LLM/embeddings/
  vector databases/agents/RAG/pipelines. Also trigger when users say things like "create a new AI project", "scaffold
  my backend", "set up my AI API", "build a RAG system", "I need an agentic workflow", "set up document processing",
  "build an AI agent", or want to combine databases with vector search and LLM capabilities. This skill handles the
  full interactive setup: language, database, AI workflow type (RAG/Agentic RAG/Custom), orchestration framework,
  chunking strategy, reranking, observability, LLM provider, ORM, authentication, and generates a complete
  enterprise-grade skeleton with production AI pipelines.
---

# GenAI Backend Setup Skill

This skill walks the user through an interactive questionnaire to configure and generate a production-ready GenAI
backend project. The generated skeleton follows enterprise-level architecture patterns used by senior ML engineers
and backend architects at scale — covering not just the API layer but the full AI pipeline: ingestion, chunking,
embedding, retrieval, reranking, generation, evaluation, and observability.

## Overview

The setup flow collects preferences through a series of questions, then generates a complete project skeleton with:
- Clean architecture (service layer, repository pattern, dependency injection)
- Role-based access control (Admin + User) with JWT and static API key auth
- Database + vector database integration (unified, not separate systems)
- Full GenAI pipeline (ingestion → chunking → embedding → retrieval → reranking → generation)
- AI workflow type: Standard RAG, Agentic RAG, or Custom workflows
- Orchestration framework integration (LangChain, LangGraph, LlamaIndex, CrewAI, or none)
- LLM provider abstraction layer with streaming support
- Observability and evaluation (tracing, cost tracking, RAGAS metrics)
- Production infrastructure (Docker, environment config, logging, error handling, health checks)

---

## Step 1: Interactive Questionnaire

Ask the following questions using `AskUserQuestion`. Ask them in batches of 2-4 questions at a time for efficient flow.
Wait for each batch response before asking the next — later questions depend on earlier answers.

### Batch 1: Language and Database

**Question 1 — Language:**
- **Python (Recommended)** — FastAPI + async. The dominant choice for GenAI backends. Mature ML ecosystem, first-class support for all embedding libraries, LLM SDKs, LangChain, LlamaIndex, and every AI framework.
- **JavaScript/TypeScript** — Fastify + TypeScript for type-safe, high-performance Node.js backends. Good when the team is full-stack JS. Note: some AI frameworks (CrewAI, LlamaIndex agents) are Python-only.

**Question 2 — Database + Vector Store:**
- **PostgreSQL + pgvector** — SQL relational database with native vector similarity search via pgvector extension. Best when you need strong relational data modeling, ACID transactions, and want vector search in the same database. Supports HNSW and IVFFlat indexes. Ideal for structured metadata + vector search together.
- **MongoDB + Atlas Vector Search** — NoSQL document database with built-in vector search via Atlas. Best for flexible schemas, document-oriented data, and when your data is naturally hierarchical/nested. Uses HNSW indexing with $vectorSearch/$rankFusion aggregation.

### Batch 2: AI Workflow Type and Orchestration

**Question 3 — AI Workflow Type:**
This is the core question that shapes the entire pipeline architecture.

- **RAG (Retrieval-Augmented Generation)** — Standard RAG pipeline: ingest documents → chunk → embed → store in vector DB → retrieve relevant chunks → generate LLM response grounded in retrieved context. Best for Q&A over documents, knowledge bases, customer support, search-augmented chat.
- **Agentic RAG** — RAG with autonomous agents that can reason, plan, use tools, and make decisions. Agents decide WHEN to retrieve, WHAT to search for, and can perform multi-step reasoning with tool calls (web search, code execution, API calls). Best for complex workflows requiring dynamic retrieval, multi-hop reasoning, and tool orchestration.
- **Custom Pipeline** — You know what you want to build and it doesn't fit the RAG/Agentic patterns. Could be document processing, content generation, classification, summarization pipelines, or something entirely novel. The skill will ask what you want to build and use web search + best practices to design the architecture.

**Question 4 — Orchestration Framework:**

Show different options based on the workflow type chosen:

If **RAG** was selected:
- **LangChain** — Most widely adopted. 70+ LLM integrations, extensive document loaders (PDF, HTML, Markdown, CSV, etc.), text splitters, vector store integrations. Best ecosystem for standard RAG. Python + JS.
- **LlamaIndex** — Purpose-built for RAG. Superior ingestion pipeline, automatic chunking optimization, built-in reranking, managed index abstractions. Best retrieval quality out of the box. Python + JS.
- **No Framework (Direct)** — Build the RAG pipeline from scratch using direct SDK calls (OpenAI, pgvector, etc.). Maximum control, zero framework overhead, but more code to write. Best when you want minimal dependencies or need non-standard pipeline behavior.

If **Agentic RAG** was selected:
- **LangGraph (Recommended)** — Graph-based agent orchestration from LangChain. Define agent workflows as directed graphs with explicit state, conditional branching, parallel execution, human-in-the-loop, and checkpointing. Most flexible and production-ready for complex agent systems. Python + JS.
- **CrewAI** — Role-based multi-agent framework. Define agents with roles, goals, and backstories. Agents collaborate on tasks using structured delegation. Higher-level abstraction, faster to prototype. **Python only**.
- **LlamaIndex Agents** — Agent framework integrated with LlamaIndex's retrieval. ReAct agents, function calling agents, and multi-agent workflows with built-in RAG tool integration. Good when retrieval is the primary agent capability. Python + JS.
- **OpenAI Agents SDK** — OpenAI's native agent framework. Tool use, handoffs between agents, guardrails, and tracing built-in. Simplest if you're committed to the OpenAI ecosystem. Python.

If **Custom Pipeline** was selected:
- Show no framework options. Instead, ask Question 6 (what to build) immediately, then do a web search for the best architecture patterns and tools for that specific use case.

### Batch 3: RAG Pipeline Configuration

Only ask these if RAG or Agentic RAG was selected. Skip for Custom Pipeline.

**Question 5 — Search / Retrieval Strategy:**
- **Vectorless (Full-text/PageIndex)** — Traditional full-text search only. Use when data is well-structured and keyword search is sufficient. PostgreSQL uses tsvector/tsquery; MongoDB uses Atlas Search.
- **Vector Only** — Pure semantic similarity search using embeddings. Best for natural language queries where meaning matters more than exact keywords.
- **Hybrid + Reranking (Recommended)** — Combines full-text + vector search using RRF (Reciprocal Rank Fusion), then reranks results with a cross-encoder for maximum precision. Industry standard for production RAG. Adds ~50-100ms latency but improves precision 10-30%.

**Question 6 — Chunking Strategy:**
- **Recursive Character (Recommended)** — Split by paragraphs → sentences → characters at 512 tokens with 10-20% overlap. Best general-purpose strategy, benchmark-validated default (69% accuracy). Zero model calls required.
- **Semantic Chunking** — Split based on embedding similarity between sentences. Groups semantically related content. ~2-3% recall improvement over recursive but requires embedding every sentence during ingestion. Higher cost.
- **Document-Aware** — Respect document structure (headings, sections, pages). Best for structured documents like PDFs, academic papers, legal docs. Uses document metadata to create meaningful boundaries.

### Batch 4: LLM Provider and Observability

**Question 7 — LLM Provider:**
- **OpenAI** — GPT-4o, GPT-4.1 family. Most widely adopted, excellent tool use and structured outputs. Best embedding models (text-embedding-3-small/large).
- **Anthropic** — Claude 4 family (Opus, Sonnet, Haiku). Strong reasoning, large context windows (up to 1M tokens), excellent for complex analysis. Uses Voyage AI for embeddings.
- **OpenRouter** — Unified API gateway to 200+ models. Best for flexibility, cost optimization, A/B testing models, and fallback routing. Pay per use across any model.

**Question 8 — Observability & Evaluation:**
- **Langfuse (Recommended)** — Open-source LLM observability. Tracing, cost tracking, prompt management, "LLM-as-a-Judge" evaluators. Self-hostable or cloud. Works with all frameworks. Best open-source option.
- **LangSmith** — LangChain's native observability platform. Deep integration with LangChain/LangGraph with automatic trace capture. Best if using LangChain ecosystem. Proprietary.
- **Arize Phoenix** — Framework-agnostic observability built on OpenTelemetry. Self-hosted, great for local debugging and production monitoring. Good if you want OpenTelemetry-native tracing.
- **None (Add Later)** — Skip observability for now. Can be added later without architecture changes.

### Batch 5: ORM and Project Details

**Question 9 — ORM / Database Client:**

Only show options that match the user's language AND database choices:

Python + PostgreSQL:
- **SQLAlchemy 2.0** — The standard Python ORM. Async via asyncpg, declarative models, Alembic migrations. Battle-tested at enterprise scale.
- **SQLModel** — Built on SQLAlchemy + Pydantic by the FastAPI creator. Less boilerplate, combines ORM models with API schemas.

Python + MongoDB:
- **Beanie** — Async ODM built on Motor + Pydantic. Native async, document validation. The go-to for async MongoDB in Python.
- **MongoEngine** — Mature synchronous ODM. Better docs but no native async.

JavaScript + PostgreSQL:
- **Prisma** — Type-safe ORM with auto-generated client, declarative schema, migrations. Most popular TypeScript ORM.
- **Drizzle** — Lightweight, SQL-like TypeScript ORM. Closer to raw SQL with full type safety. Better performance.

JavaScript + MongoDB:
- **Mongoose** — The standard MongoDB ODM for Node.js. Schema validation, middleware, population.
- **Prisma** — Same Prisma DX for MongoDB. Type-safe, but MongoDB support less mature.

**Question 10 — What will your AI backend do?**

Free-text question. Ask the user to describe their specific use case. Examples:
- "A RAG-powered customer support API that answers questions from our knowledge base"
- "An AI document processing pipeline that extracts and summarizes legal contracts"
- "A multi-agent research assistant that searches the web and synthesizes reports"
- "An intelligent product search API with natural language queries"
- "A code review agent that analyzes PRs and suggests improvements"

Use this answer to customize: domain models, API endpoints, agent tools, pipeline stages, and the README.

---

## Step 2: Load Reference Files

Based on the user's choices, read the appropriate reference files from the `references/` directory:

| Choice | Reference File |
|--------|---------------|
| Python + PostgreSQL | `references/python-sql.md` |
| Python + MongoDB | `references/python-nosql.md` |
| JavaScript + PostgreSQL | `references/js-sql.md` |
| JavaScript + MongoDB | `references/js-nosql.md` |
| RAG pipeline | `references/rag-pipeline.md` |
| Agentic RAG | `references/agentic-rag.md` |
| Custom Pipeline | Do a **web search** for best architecture patterns for the user's specific use case |
| Observability | `references/observability.md` |
| Auth (always) | `references/auth-patterns.md` |
| Architecture (always) | `references/architecture.md` |

Read the stack-specific reference (1 file), the pipeline reference (1 file based on workflow type), observability (if selected), plus BOTH auth and architecture references. For Custom Pipeline, use `WebSearch` to find the best patterns for what the user described.

---

## Step 3: Present the Plan

Before generating any code, present a clear plan to the user that includes:

1. **Tech Stack Summary** — All chosen technologies with versions
2. **Architecture Diagram** (ASCII) — Show the pipeline flow visually:
   ```
   User Request → API Gateway → Auth → Pipeline Router
                                        ├→ Ingestion Pipeline: Upload → Parse → Chunk → Embed → Store
                                        ├→ Retrieval Pipeline: Query → Embed → Search → Rerank → Context
                                        └→ Generation Pipeline: Context + Prompt → LLM → Response
   ```
3. **Project Structure** — The directory tree that will be generated
4. **Pipeline Design** — How ingestion, retrieval, and generation work end-to-end
5. **Agent Design** (if Agentic) — Agent roles, tools, state graph, decision points
6. **API Endpoints** — Auth endpoints + domain endpoints + pipeline endpoints
7. **Database Schema** — Auth tables + document/chunk storage + conversation history
8. **Observability** — What gets traced, where metrics go, how to evaluate
9. **Infrastructure** — Docker setup, environment configuration, health checks

Ask the user to confirm or request changes before proceeding.

---

## Step 4: Generate the Skeleton

Generate the complete project skeleton following these principles:

### Enterprise Architecture Principles

1. **Layered Architecture** — Routes/controllers → services → repositories → models. Each layer only talks to adjacent layers.

2. **Repository Pattern** — All database access through repository classes. Never put queries in route handlers or services.

3. **Service Layer** — Business logic lives in services. Services orchestrate between repositories, LLM providers, pipelines, and external systems. Route handlers are thin.

4. **Pipeline Pattern** — AI pipelines are first-class citizens with their own module. Each pipeline stage (parse, chunk, embed, retrieve, rerank, generate) is an independent, composable unit that can be tested and swapped.

5. **Dependency Injection** — Constructor injection (Python: FastAPI `Depends()`, JS: manual DI). Enables testing and swapping implementations.

6. **Configuration Management** — All config from environment variables via validated config module. Never hardcode secrets, URLs, model names, or chunk sizes.

7. **Error Handling** — Global error handler. Custom exceptions for domain errors. Structured error responses with request ID.

8. **Logging** — Structured JSON logging (Python: structlog, JS: pino). Request ID tracking. Log levels configurable per environment.

9. **Health Checks** — `/health` endpoint checking: database, vector index, LLM provider, embedding service, observability backend.

10. **Input Validation** — Validate all inputs at boundary. Python: Pydantic. JS: Zod.

### What to Generate

Generate ALL of the following with real, working code — not placeholder comments:

**Core Infrastructure:**
- Entry point — Application factory, middleware, route mounting
- Config — Environment variables, validation, typed config
- Database — Connection, session/client management, migrations
- Models/Schemas — User, Role, Permission, ApiKey + domain models
- Auth module — JWT, API keys, RBAC middleware, password hashing
- Middleware — Error handler, logging, CORS, rate limiting
- Docker — Dockerfile, docker-compose.yml with app + database + any required services

**AI Pipeline:**
- Document Ingestion — File upload endpoint, document parsing (PDF, text, markdown), metadata extraction
- Chunking — Configurable chunking strategy (recursive, semantic, or document-aware based on user choice)
- Embedding — Embedding service with provider abstraction, batch processing, retry logic
- Vector Storage — Store/update/delete embeddings in the chosen vector database
- Retrieval — Search service implementing the chosen strategy (vectorless/vector/hybrid)
- Reranking — Cross-encoder reranking stage (if hybrid was chosen), with fallback to no reranking
- Context Assembly — Build the LLM context window from retrieved chunks with source attribution
- Generation — LLM service with streaming support, system prompts, conversation history

**If Agentic RAG:**
- Agent definitions — Agent classes/functions with roles, tools, and state
- Tool registry — Tools the agent can use (search, calculate, web fetch, etc.)
- State management — Agent state graph, checkpointing, conversation memory
- Orchestration — Agent execution loop, tool dispatch, result aggregation

**Observability (if selected):**
- Tracing integration — Instrument pipelines with trace spans
- Cost tracking — Token usage logging per request
- Evaluation hooks — RAGAS-style metrics (faithfulness, relevance, context precision)

**Supporting Files:**
- Repositories — Database access layer for all models
- Services — Auth, LLM, embedding, search, ingestion, and domain services
- Routes — Auth + domain + pipeline endpoints with validation
- Environment — .env.example with all variables documented
- Dependencies — Pinned versions in requirements.txt/pyproject.toml or package.json
- Seed script — Create initial admin, roles, permissions
- README — Architecture overview, setup instructions, API docs, pipeline documentation

### Code Quality Standards

- Type hints everywhere (Python annotations, TypeScript strict mode)
- Async/await for all I/O operations
- No `any` types in TypeScript, no untyped functions in Python
- Constants and enums for roles, permissions, pipeline stages, status codes
- Meaningful names — self-documenting code
- Security: parameterized queries, bcrypt passwords, secure JWT config, CORS whitelist, rate limiting
- No secrets in code — everything from environment variables
- Pipeline stages are independently testable and composable

---

## Step 5: Post-Generation Guidance

After generating the skeleton, provide:

1. **Quick Start** — Exact commands: docker-compose up, install deps, run migrations, seed DB, start dev server
2. **Ingest Your First Document** — curl example to upload a document and see it get chunked + embedded
3. **Ask Your First Question** — curl example showing the full RAG pipeline (query → retrieve → generate)
4. **Test Auth** — curl examples for register, login, JWT token, API key creation
5. **View Traces** (if observability) — How to access the tracing dashboard
6. **Next Steps** — Priority-ordered list based on their stated goal
7. **Production Checklist** — What to change before deploying (secrets, CORS, rate limits, model selection, connection pooling)
8. **Scaling Considerations** — Background workers for ingestion, caching embeddings, horizontal scaling, queue-based processing

---

## Important Notes

- Always use the LATEST stable versions of all dependencies. Check reference files for current versions.
- Generated code must actually work — no placeholder `pass` statements or `// TODO` in critical paths.
- Security is non-negotiable: never skip password hashing, JWT validation, input sanitization.
- LLM provider layer must be abstract — switching providers = changing config, not code.
- Pipeline stages must be behind interfaces — swappable chunking, embedding, retrieval, and reranking strategies.
- Include seed script for initial admin user and roles.
- For Custom Pipeline: always do a web search to find the best current architecture patterns before generating code.
- If the user chose a framework (LangChain/LangGraph/LlamaIndex/CrewAI), use that framework's idioms and patterns — don't fight the framework.
- If the user chose "No Framework", build everything from direct SDK calls with clean abstractions.
