# Python + PostgreSQL + pgvector Stack Reference

## Table of Contents
1. [Dependencies](#dependencies)
2. [Project Structure](#project-structure)
3. [Database Setup](#database-setup)
4. [ORM Configuration](#orm-configuration)
5. [Vector Search Setup](#vector-search-setup)
6. [Search Implementations](#search-implementations)
7. [FastAPI Application Factory](#fastapi-application-factory)

---

## Dependencies

### Core (pyproject.toml or requirements.txt)

```
# Web Framework
fastapi>=0.115.0
uvicorn[standard]>=0.34.0
pydantic>=2.10.0
pydantic-settings>=2.7.0

# Database
asyncpg>=0.30.0
sqlalchemy[asyncio]>=2.0.36
alembic>=1.14.0
pgvector>=0.3.6

# Auth
python-jose[cryptography]>=3.3.0
passlib[bcrypt]>=1.7.4
python-multipart>=0.0.18

# LLM / Embeddings
openai>=1.60.0
anthropic>=0.42.0
tiktoken>=0.8.0

# Utilities
structlog>=24.4.0
httpx>=0.28.0
tenacity>=9.0.0
python-dotenv>=1.0.1

# Dev
pytest>=8.3.0
pytest-asyncio>=0.25.0
ruff>=0.8.0
mypy>=1.14.0
```

### SQLModel variant (replace SQLAlchemy lines)
```
sqlmodel>=0.0.22
asyncpg>=0.30.0
alembic>=1.14.0
```

---

## Project Structure

```
project_name/
├── app/
│   ├── __init__.py
│   ├── main.py                    # Application factory
│   ├── config.py                  # Settings from env vars
│   ├── dependencies.py            # FastAPI dependency injection
│   │
│   ├── db/
│   │   ├── __init__.py
│   │   ├── session.py             # Async engine + session factory
│   │   ├── base.py                # Declarative base
│   │   └── migrations/            # Alembic migrations
│   │       ├── env.py
│   │       ├── alembic.ini
│   │       └── versions/
│   │
│   ├── models/
│   │   ├── __init__.py
│   │   ├── user.py                # User, Role, Permission models
│   │   ├── api_key.py             # Static API key model
│   │   └── document.py            # Domain model (example)
│   │
│   ├── schemas/
│   │   ├── __init__.py
│   │   ├── auth.py                # Login, Token, Register schemas
│   │   ├── user.py                # User CRUD schemas
│   │   └── document.py            # Domain schemas
│   │
│   ├── repositories/
│   │   ├── __init__.py
│   │   ├── base.py                # Generic async repository
│   │   ├── user_repository.py
│   │   ├── api_key_repository.py
│   │   └── document_repository.py
│   │
│   ├── services/
│   │   ├── __init__.py
│   │   ├── auth_service.py        # JWT + API key auth logic
│   │   ├── user_service.py
│   │   ├── llm_service.py         # LLM provider abstraction
│   │   ├── embedding_service.py   # Embedding generation
│   │   ├── search_service.py      # Vector/hybrid/fulltext search
│   │   └── document_service.py    # Domain service
│   │
│   ├── api/
│   │   ├── __init__.py
│   │   ├── v1/
│   │   │   ├── __init__.py
│   │   │   ├── router.py          # Aggregates all v1 routes
│   │   │   ├── auth.py            # /auth/login, /auth/register
│   │   │   ├── users.py           # /users CRUD
│   │   │   └── documents.py       # Domain routes
│   │   └── health.py              # /health endpoint
│   │
│   ├── middleware/
│   │   ├── __init__.py
│   │   ├── error_handler.py       # Global exception handler
│   │   ├── logging_middleware.py   # Request/response logging
│   │   └── rate_limiter.py        # Rate limiting
│   │
│   ├── core/
│   │   ├── __init__.py
│   │   ├── security.py            # Password hashing, JWT utils
│   │   ├── permissions.py         # RBAC decorators and checks
│   │   ├── exceptions.py          # Custom exception classes
│   │   └── constants.py           # Roles, permissions enums
│   │
│   └── llm/
│       ├── __init__.py
│       ├── base.py                # Abstract LLM provider interface
│       ├── openai_provider.py
│       ├── anthropic_provider.py
│       └── openrouter_provider.py
│
├── scripts/
│   └── seed.py                    # Create initial admin + roles
│
├── tests/
│   ├── conftest.py                # Fixtures, test DB setup
│   ├── test_auth.py
│   └── test_documents.py
│
├── Dockerfile
├── docker-compose.yml
├── .env.example
├── pyproject.toml
└── README.md
```

---

## Database Setup

### Async Engine + Session (db/session.py)

```python
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine
from app.config import settings

engine = create_async_engine(
    settings.database_url,
    echo=settings.debug,
    pool_size=20,
    max_overflow=10,
    pool_pre_ping=True,
    pool_recycle=300,
)

async_session_factory = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,
)

async def get_session() -> AsyncSession:
    async with async_session_factory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

### Enable pgvector extension

In Alembic migration or startup:
```python
from sqlalchemy import text

async def init_extensions(engine):
    async with engine.begin() as conn:
        await conn.execute(text("CREATE EXTENSION IF NOT EXISTS vector"))
```

---

## ORM Configuration

### SQLAlchemy 2.0 Models

```python
from sqlalchemy import Column, String, Integer, ForeignKey, DateTime, Boolean, Text, Index
from sqlalchemy.orm import relationship, Mapped, mapped_column
from pgvector.sqlalchemy import Vector
from app.db.base import Base
import datetime

class Document(Base):
    __tablename__ = "documents"

    id: Mapped[int] = mapped_column(primary_key=True, index=True)
    title: Mapped[str] = mapped_column(String(500))
    content: Mapped[str] = mapped_column(Text)
    embedding: Mapped[list[float]] = mapped_column(Vector(1536), nullable=True)
    metadata_json: Mapped[dict | None] = mapped_column(JSON, nullable=True)
    owner_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    created_at: Mapped[datetime.datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now()
    )

    # HNSW index for vector similarity search
    __table_args__ = (
        Index(
            "ix_documents_embedding_hnsw",
            embedding,
            postgresql_using="hnsw",
            postgresql_with={"m": 16, "ef_construction": 64},
            postgresql_ops={"embedding": "vector_cosine_ops"},
        ),
    )
```

### SQLModel Variant

```python
from sqlmodel import SQLModel, Field
from pgvector.sqlalchemy import Vector
from sqlalchemy import Column

class Document(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    title: str = Field(max_length=500)
    content: str
    embedding: list[float] | None = Field(
        default=None, sa_column=Column(Vector(1536))
    )
    owner_id: int = Field(foreign_key="users.id")
```

---

## Vector Search Setup

### Embedding Generation

```python
from abc import ABC, abstractmethod
import numpy as np

class EmbeddingProvider(ABC):
    @abstractmethod
    async def embed(self, text: str) -> list[float]: ...

    @abstractmethod
    async def embed_batch(self, texts: list[str]) -> list[list[float]]: ...

class OpenAIEmbedding(EmbeddingProvider):
    def __init__(self, client, model: str = "text-embedding-3-small"):
        self.client = client
        self.model = model

    async def embed(self, text: str) -> list[float]:
        response = await self.client.embeddings.create(input=text, model=self.model)
        return response.data[0].embedding

    async def embed_batch(self, texts: list[str]) -> list[list[float]]:
        response = await self.client.embeddings.create(input=texts, model=self.model)
        return [item.embedding for item in response.data]
```

---

## Search Implementations

### Vectorless (Full-Text Only)

```python
from sqlalchemy import func, text

async def fulltext_search(session, query: str, limit: int = 10):
    ts_query = func.plainto_tsquery("english", query)
    stmt = (
        select(Document)
        .where(func.to_tsvector("english", Document.content).op("@@")(ts_query))
        .order_by(func.ts_rank(func.to_tsvector("english", Document.content), ts_query).desc())
        .limit(limit)
    )
    result = await session.execute(stmt)
    return result.scalars().all()
```

**Add a GIN index for full-text search performance:**
```sql
CREATE INDEX ix_documents_content_fts ON documents USING gin(to_tsvector('english', content));
```

### Vector Only

```python
async def vector_search(session, query_embedding: list[float], limit: int = 10):
    stmt = (
        select(Document)
        .where(Document.embedding.isnot(None))
        .order_by(Document.embedding.cosine_distance(query_embedding))
        .limit(limit)
    )
    result = await session.execute(stmt)
    return result.scalars().all()
```

### Hybrid (RRF Fusion)

```python
async def hybrid_search(
    session,
    query: str,
    query_embedding: list[float],
    limit: int = 10,
    rrf_k: int = 60,
    semantic_weight: float = 0.5,
):
    """Reciprocal Rank Fusion combining full-text and vector search."""
    # Get full-text results with rank
    fts_results = await fulltext_search(session, query, limit=limit * 2)
    vec_results = await vector_search(session, query_embedding, limit=limit * 2)

    # Build RRF scores
    scores: dict[int, float] = {}
    text_weight = 1.0 - semantic_weight

    for rank, doc in enumerate(fts_results):
        scores[doc.id] = scores.get(doc.id, 0) + text_weight / (rrf_k + rank + 1)

    for rank, doc in enumerate(vec_results):
        scores[doc.id] = scores.get(doc.id, 0) + semantic_weight / (rrf_k + rank + 1)

    # Sort by combined score and return top results
    sorted_ids = sorted(scores, key=scores.get, reverse=True)[:limit]

    stmt = select(Document).where(Document.id.in_(sorted_ids))
    result = await session.execute(stmt)
    docs = {doc.id: doc for doc in result.scalars().all()}
    return [docs[doc_id] for doc_id in sorted_ids if doc_id in docs]
```

---

## FastAPI Application Factory

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from app.config import settings
from app.db.session import engine, init_extensions
from app.api.v1.router import api_v1_router
from app.api.health import health_router
from app.middleware.error_handler import register_exception_handlers
from app.middleware.logging_middleware import LoggingMiddleware

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    await init_extensions(engine)
    yield
    # Shutdown
    await engine.dispose()

def create_app() -> FastAPI:
    app = FastAPI(
        title=settings.app_name,
        version=settings.app_version,
        docs_url="/api/docs" if settings.debug else None,
        lifespan=lifespan,
    )

    app.add_middleware(
        CORSMiddleware,
        allow_origins=settings.cors_origins,
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )
    app.add_middleware(LoggingMiddleware)

    register_exception_handlers(app)

    app.include_router(health_router)
    app.include_router(api_v1_router, prefix="/api/v1")

    return app

app = create_app()
```

---

## Docker Compose

```yaml
services:
  app:
    build: .
    ports:
      - "${APP_PORT:-8000}:8000"
    env_file: .env
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - .:/app
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload

  db:
    image: pgvector/pgvector:pg17
    environment:
      POSTGRES_USER: ${DB_USER:-postgres}
      POSTGRES_PASSWORD: ${DB_PASSWORD:-postgres}
      POSTGRES_DB: ${DB_NAME:-ai_backend}
    ports:
      - "${DB_PORT:-5432}:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  pgdata:
```

## .env.example

```bash
# Application
APP_NAME=ai-backend
APP_VERSION=0.1.0
APP_ENV=development
DEBUG=true
APP_PORT=8000

# Database
DATABASE_URL=postgresql+asyncpg://postgres:postgres@db:5432/ai_backend
DB_USER=postgres
DB_PASSWORD=postgres
DB_NAME=ai_backend
DB_PORT=5432

# Auth
JWT_SECRET_KEY=CHANGE_ME_TO_A_RANDOM_SECRET
JWT_ALGORITHM=HS256
JWT_ACCESS_TOKEN_EXPIRE_MINUTES=30
JWT_REFRESH_TOKEN_EXPIRE_DAYS=7

# LLM Provider
LLM_PROVIDER=openai  # openai | anthropic | openrouter
OPENAI_API_KEY=
ANTHROPIC_API_KEY=
OPENROUTER_API_KEY=

# Embedding
EMBEDDING_MODEL=text-embedding-3-small
EMBEDDING_DIMENSIONS=1536

# CORS
CORS_ORIGINS=["http://localhost:3000"]

# Logging
LOG_LEVEL=INFO
```
