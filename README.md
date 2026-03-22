# Agentic Backend Skill

A production-ready skill that interactively scaffolds enterprise-grade GenAI backends. Works with **Claude Code**, **Cursor**, and **OpenAI Codex**.

It walks you through choosing your stack — language, database + vector DB, AI workflow (RAG / Agentic RAG / Custom), orchestration framework, chunking strategy, reranking, observability, LLM provider, ORM, and auth — then generates a complete working skeleton.

## What It Generates

- Full AI pipeline: ingestion → chunking → embedding → retrieval → reranking → generation
- Role-based auth (Admin/User) with JWT + static API keys
- LLM provider abstraction (OpenAI, Anthropic, OpenRouter — swap via config)
- Observability & evaluation (Langfuse, LangSmith, or Arize Phoenix)
- Docker, structured logging, health checks, environment config
- Production-ready project structure following clean architecture

## Supported Stacks

| | Python | JavaScript/TypeScript |
|---|--------|----------------------|
| **Framework** | FastAPI | Fastify |
| **SQL + Vector** | PostgreSQL + pgvector | PostgreSQL + pgvector |
| **NoSQL + Vector** | MongoDB + Atlas Vector Search | MongoDB + Atlas Vector Search |
| **ORMs** | SQLAlchemy, SQLModel, Beanie, MongoEngine | Prisma, Drizzle, Mongoose |
| **AI Frameworks** | LangChain, LangGraph, LlamaIndex, CrewAI | LangChain, LangGraph, LlamaIndex |
| **Search** | Full-text, Vector, Hybrid (RRF + Reranking) | Full-text, Vector, Hybrid (RRF + Reranking) |

---

## Installation

### Claude Code

**Option 1 — Copy the skill folder into your project:**

```bash
# Clone the repo
git clone git@github.com:Mayureshju/agentic-backend-skill.git /tmp/agentic-backend-skill

# Copy the skill into your project
mkdir -p .agents/skills
cp -r /tmp/agentic-backend-skill/.agents/skills/ai-backend-setup .agents/skills/

# Clean up
rm -rf /tmp/agentic-backend-skill
```

**Option 2 — Install globally (available in all projects):**

```bash
git clone git@github.com:Mayureshju/agentic-backend-skill.git /tmp/agentic-backend-skill

mkdir -p ~/.claude/skills
cp -r /tmp/agentic-backend-skill/.agents/skills/ai-backend-setup ~/.claude/skills/

rm -rf /tmp/agentic-backend-skill
```

Once installed, the skill triggers automatically when you ask Claude Code to set up an AI backend, build a RAG system, create an agentic workflow, etc.

---

### Cursor

Cursor uses **Project Rules** (`.cursor/rules/`) for persistent instructions.

```bash
# Clone the repo
git clone git@github.com:Mayureshju/agentic-backend-skill.git /tmp/agentic-backend-skill

# Create rules directory
mkdir -p .cursor/rules

# Copy the SKILL.md as a Cursor rule
cp /tmp/agentic-backend-skill/.agents/skills/ai-backend-setup/SKILL.md .cursor/rules/ai-backend-setup.md

# Copy reference files so the AI can read them
mkdir -p .cursor/rules/ai-backend-references
cp /tmp/agentic-backend-skill/.agents/skills/ai-backend-setup/references/*.md .cursor/rules/ai-backend-references/

# Clean up
rm -rf /tmp/agentic-backend-skill
```

Then in Cursor, the AI will follow these instructions when you ask it to set up an AI backend.

> **Tip:** You can also paste the content of `SKILL.md` into Cursor's **Settings → Rules → User Rules** for global access across all projects.

---

### OpenAI Codex

Codex reads skills from `.agents/skills/` — the same structure as this repo.

```bash
# Clone the repo
git clone git@github.com:Mayureshju/agentic-backend-skill.git /tmp/agentic-backend-skill

# Copy into your project (Codex scans .agents/skills/ in project root)
mkdir -p .agents/skills
cp -r /tmp/agentic-backend-skill/.agents/skills/ai-backend-setup .agents/skills/

# Clean up
rm -rf /tmp/agentic-backend-skill
```

**Or install globally (available in all projects):**

```bash
mkdir -p ~/.agents/skills
cp -r /tmp/agentic-backend-skill/.agents/skills/ai-backend-setup ~/.agents/skills/
```

Restart Codex after installing. The skill will be available when you ask Codex to build an AI backend.

---

## Quick One-Liner Install

```bash
# Claude Code (project-level)
git clone git@github.com:Mayureshju/agentic-backend-skill.git /tmp/_skill && mkdir -p .agents/skills && cp -r /tmp/_skill/.agents/skills/ai-backend-setup .agents/skills/ && rm -rf /tmp/_skill

# Codex (project-level) — same command, same structure
git clone git@github.com:Mayureshju/agentic-backend-skill.git /tmp/_skill && mkdir -p .agents/skills && cp -r /tmp/_skill/.agents/skills/ai-backend-setup .agents/skills/ && rm -rf /tmp/_skill

# Cursor (project-level)
git clone git@github.com:Mayureshju/agentic-backend-skill.git /tmp/_skill && mkdir -p .cursor/rules/ai-backend-references && cp /tmp/_skill/.agents/skills/ai-backend-setup/SKILL.md .cursor/rules/ai-backend-setup.md && cp /tmp/_skill/.agents/skills/ai-backend-setup/references/*.md .cursor/rules/ai-backend-references/ && rm -rf /tmp/_skill
```

---

## Usage

Just ask your AI coding assistant to set up a GenAI backend. Example prompts:

- *"Set up a RAG backend with Python, PostgreSQL, and LangChain"*
- *"Create an agentic AI backend with TypeScript and MongoDB"*
- *"Build me a production AI API with hybrid search and observability"*
- *"Scaffold a document Q&A system with authentication"*

The skill will ask you a series of questions to configure the stack, then generate the complete project.

---

## Skill Structure

```
.agents/skills/ai-backend-setup/
├── SKILL.md                          # Main skill (interactive questionnaire + generation instructions)
└── references/
    ├── python-sql.md                 # Python + PostgreSQL + pgvector
    ├── python-nosql.md               # Python + MongoDB + Atlas Vector Search
    ├── js-sql.md                     # TypeScript + PostgreSQL + pgvector
    ├── js-nosql.md                   # TypeScript + MongoDB + Atlas Vector Search
    ├── rag-pipeline.md               # RAG pipeline: ingestion, chunking, reranking, generation
    ├── agentic-rag.md                # LangGraph, CrewAI, LlamaIndex Agents, OpenAI Agents SDK
    ├── auth-patterns.md              # RBAC, JWT, static API keys
    ├── observability.md              # Langfuse, LangSmith, Arize Phoenix, RAGAS evaluation
    └── architecture.md               # Enterprise patterns, LLM abstraction, repository pattern
```

---

## License

MIT
