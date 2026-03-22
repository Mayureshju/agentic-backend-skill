# Python + MongoDB + Atlas Vector Search Stack Reference

## Table of Contents
1. [Dependencies](#dependencies)
2. [Project Structure](#project-structure)
3. [Database Setup](#database-setup)
4. [ODM Configuration](#odm-configuration)
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
motor>=3.7.0
beanie>=1.27.0
pymongo>=4.10.0

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

### MongoEngine variant (replace motor/beanie)
```
mongoengine>=0.29.0
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
│   │   ├── mongodb.py             # Motor client + Beanie init
│   │   └── indexes.py             # Vector + text index definitions
│   │
│   ├── models/
│   │   ├── __init__.py
│   │   ├── user.py                # User document with roles
│   │   ├── api_key.py             # API key document
│   │   └── document.py            # Domain document model
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
│   │   ├── auth_service.py
│   │   ├── user_service.py
│   │   ├── llm_service.py
│   │   ├── embedding_service.py
│   │   ├── search_service.py
│   │   └── document_service.py
│   │
│   ├── api/
│   │   ├── __init__.py
│   │   ├── v1/
│   │   │   ├── __init__.py
│   │   │   ├── router.py
│   │   │   ├── auth.py
│   │   │   ├── users.py
│   │   │   └── documents.py
│   │   └── health.py
│   │
│   ├── middleware/
│   │   ├── __init__.py
│   │   ├── error_handler.py
│   │   ├── logging_middleware.py
│   │   └── rate_limiter.py
│   │
│   ├── core/
│   │   ├── __init__.py
│   │   ├── security.py
│   │   ├── permissions.py
│   │   ├── exceptions.py
│   │   └── constants.py
│   │
│   └── llm/
│       ├── __init__.py
│       ├── base.py
│       ├── openai_provider.py
│       ├── anthropic_provider.py
│       └── openrouter_provider.py
│
├── scripts/
│   └── seed.py
│
├── tests/
│   ├── conftest.py
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

### Motor Client + Beanie Init (db/mongodb.py)

```python
from motor.motor_asyncio import AsyncIOMotorClient
from beanie import init_beanie
from app.config import settings
from app.models.user import User
from app.models.api_key import ApiKey
from app.models.document import Document

client: AsyncIOMotorClient | None = None

async def init_db():
    global client
    client = AsyncIOMotorClient(settings.mongodb_url)
    database = client[settings.db_name]

    await init_beanie(
        database=database,
        document_models=[User, ApiKey, Document],
    )

async def close_db():
    global client
    if client:
        client.close()

def get_database():
    return client[settings.db_name]
```

---

## ODM Configuration

### Beanie Document Models

```python
from beanie import Document as BeanieDocument, Indexed
from pydantic import Field
from datetime import datetime
from typing import Optional

class Document(BeanieDocument):
    title: Indexed(str)
    content: str
    embedding: Optional[list[float]] = None
    metadata: dict = Field(default_factory=dict)
    owner_id: str
    created_at: datetime = Field(default_factory=datetime.utcnow)
    updated_at: datetime = Field(default_factory=datetime.utcnow)

    class Settings:
        name = "documents"
        use_state_management = True
        indexes = [
            "title",
            "owner_id",
            [("created_at", -1)],
        ]
```

### MongoEngine Variant

```python
from mongoengine import Document, StringField, ListField, FloatField, DictField, DateTimeField, ReferenceField
from datetime import datetime

class DocumentModel(Document):
    title = StringField(required=True, max_length=500)
    content = StringField(required=True)
    embedding = ListField(FloatField())
    metadata = DictField()
    owner = ReferenceField("User")
    created_at = DateTimeField(default=datetime.utcnow)

    meta = {
        "collection": "documents",
        "indexes": ["title", "owner", "-created_at"],
    }
```

---

## Vector Search Setup

### Atlas Vector Search Index

Create the vector search index via Atlas UI, Atlas CLI, or programmatically:

```json
{
  "fields": [
    {
      "type": "vector",
      "path": "embedding",
      "numDimensions": 1536,
      "similarity": "cosine"
    },
    {
      "type": "filter",
      "path": "owner_id"
    },
    {
      "type": "filter",
      "path": "metadata.category"
    }
  ]
}
```

### Atlas Search Index (for full-text)

```json
{
  "mappings": {
    "dynamic": false,
    "fields": {
      "title": { "type": "string", "analyzer": "lucene.standard" },
      "content": { "type": "string", "analyzer": "lucene.standard" }
    }
  }
}
```

### Programmatic Index Creation (db/indexes.py)

```python
from pymongo.operations import SearchIndexModel

async def create_vector_index(database):
    collection = database["documents"]
    vector_index = SearchIndexModel(
        definition={
            "fields": [
                {
                    "type": "vector",
                    "path": "embedding",
                    "numDimensions": 1536,
                    "similarity": "cosine",
                },
                {"type": "filter", "path": "owner_id"},
            ]
        },
        name="vector_index",
        type="vectorSearch",
    )
    await collection.create_search_index(model=vector_index)

async def create_search_index(database):
    collection = database["documents"]
    search_index = SearchIndexModel(
        definition={
            "mappings": {
                "dynamic": False,
                "fields": {
                    "title": {"type": "string", "analyzer": "lucene.standard"},
                    "content": {"type": "string", "analyzer": "lucene.standard"},
                },
            }
        },
        name="search_index",
        type="search",
    )
    await collection.create_search_index(model=search_index)
```

---

## Search Implementations

### Vectorless (Atlas Full-Text Search)

```python
async def fulltext_search(database, query: str, limit: int = 10, owner_id: str | None = None):
    collection = database["documents"]
    pipeline = [
        {
            "$search": {
                "index": "search_index",
                "text": {
                    "query": query,
                    "path": ["title", "content"],
                },
            }
        },
        {"$addFields": {"search_score": {"$meta": "searchScore"}}},
    ]
    if owner_id:
        pipeline.append({"$match": {"owner_id": owner_id}})
    pipeline.append({"$limit": limit})

    cursor = collection.aggregate(pipeline)
    return await cursor.to_list(length=limit)
```

### Vector Only

```python
async def vector_search(
    database,
    query_embedding: list[float],
    limit: int = 10,
    owner_id: str | None = None,
):
    collection = database["documents"]
    pipeline = [
        {
            "$vectorSearch": {
                "index": "vector_index",
                "path": "embedding",
                "queryVector": query_embedding,
                "numCandidates": limit * 10,
                "limit": limit,
            }
        },
        {"$addFields": {"vector_score": {"$meta": "vectorSearchScore"}}},
    ]
    if owner_id:
        pipeline[0]["$vectorSearch"]["filter"] = {"owner_id": owner_id}

    cursor = collection.aggregate(pipeline)
    return await cursor.to_list(length=limit)
```

### Hybrid (RRF via $rankFusion or manual)

```python
async def hybrid_search(
    database,
    query: str,
    query_embedding: list[float],
    limit: int = 10,
    owner_id: str | None = None,
):
    """
    Hybrid search using manual RRF. For Atlas clusters that support $rankFusion,
    you can use the native aggregation stage instead.
    """
    fts_results = await fulltext_search(database, query, limit=limit * 2, owner_id=owner_id)
    vec_results = await vector_search(database, query_embedding, limit=limit * 2, owner_id=owner_id)

    # Reciprocal Rank Fusion
    rrf_k = 60
    scores: dict[str, float] = {}
    doc_map: dict[str, dict] = {}

    for rank, doc in enumerate(fts_results):
        doc_id = str(doc["_id"])
        scores[doc_id] = scores.get(doc_id, 0) + 0.5 / (rrf_k + rank + 1)
        doc_map[doc_id] = doc

    for rank, doc in enumerate(vec_results):
        doc_id = str(doc["_id"])
        scores[doc_id] = scores.get(doc_id, 0) + 0.5 / (rrf_k + rank + 1)
        doc_map[doc_id] = doc

    sorted_ids = sorted(scores, key=scores.get, reverse=True)[:limit]
    return [doc_map[doc_id] for doc_id in sorted_ids]
```

#### Native $rankFusion (Atlas 8.0+)

```python
async def hybrid_search_native(database, query: str, query_embedding: list[float], limit: int = 10):
    """Uses MongoDB Atlas native $rankFusion for hybrid search."""
    collection = database["documents"]
    pipeline = [
        {
            "$rankFusion": {
                "input": {
                    "pipelines": {
                        "vector": [
                            {
                                "$vectorSearch": {
                                    "index": "vector_index",
                                    "path": "embedding",
                                    "queryVector": query_embedding,
                                    "numCandidates": limit * 10,
                                    "limit": limit * 2,
                                }
                            }
                        ],
                        "text": [
                            {
                                "$search": {
                                    "index": "search_index",
                                    "text": {"query": query, "path": ["title", "content"]},
                                }
                            },
                            {"$limit": limit * 2},
                        ],
                    }
                }
            }
        },
        {"$limit": limit},
    ]
    cursor = collection.aggregate(pipeline)
    return await cursor.to_list(length=limit)
```

---

## FastAPI Application Factory

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from app.config import settings
from app.db.mongodb import init_db, close_db
from app.api.v1.router import api_v1_router
from app.api.health import health_router
from app.middleware.error_handler import register_exception_handlers
from app.middleware.logging_middleware import LoggingMiddleware

@asynccontextmanager
async def lifespan(app: FastAPI):
    await init_db()
    yield
    await close_db()

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
      mongodb:
        condition: service_healthy
    volumes:
      - .:/app
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload

  mongodb:
    image: mongodb/mongodb-atlas-local:8.0
    environment:
      MONGODB_INITDB_ROOT_USERNAME: ${MONGO_USER:-admin}
      MONGODB_INITDB_ROOT_PASSWORD: ${MONGO_PASSWORD:-admin}
    ports:
      - "${MONGO_PORT:-27017}:27017"
    volumes:
      - mongodata:/data/db
    healthcheck:
      test: ["CMD", "mongosh", "--eval", "db.adminCommand('ping')"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  mongodata:
```

**Note:** The `mongodb/mongodb-atlas-local` image supports Atlas Search and Atlas Vector Search locally. For production, use MongoDB Atlas (cloud).

## .env.example

```bash
# Application
APP_NAME=ai-backend
APP_VERSION=0.1.0
APP_ENV=development
DEBUG=true
APP_PORT=8000

# MongoDB
MONGODB_URL=mongodb://admin:admin@mongodb:27017/ai_backend?authSource=admin
DB_NAME=ai_backend
MONGO_USER=admin
MONGO_PASSWORD=admin
MONGO_PORT=27017

# Auth
JWT_SECRET_KEY=CHANGE_ME_TO_A_RANDOM_SECRET
JWT_ALGORITHM=HS256
JWT_ACCESS_TOKEN_EXPIRE_MINUTES=30
JWT_REFRESH_TOKEN_EXPIRE_DAYS=7

# LLM Provider
LLM_PROVIDER=openai
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
