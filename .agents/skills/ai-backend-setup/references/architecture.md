# Enterprise Architecture Patterns Reference

## Table of Contents
1. [Layered Architecture](#layered-architecture)
2. [LLM Provider Abstraction](#llm-provider-abstraction)
3. [Repository Pattern](#repository-pattern)
4. [Error Handling](#error-handling)
5. [Logging & Observability](#logging--observability)
6. [Configuration Management](#configuration-management)
7. [Health Checks](#health-checks)
8. [Docker & Infrastructure](#docker--infrastructure)
9. [Testing Strategy](#testing-strategy)
10. [API Design](#api-design)

---

## Layered Architecture

```
┌─────────────────────────────────────┐
│          Routes / Controllers       │  ← HTTP layer: validation, auth, response formatting
├─────────────────────────────────────┤
│          Services                   │  ← Business logic: orchestration, rules, workflows
├─────────────────────────────────────┤
│          Repositories               │  ← Data access: queries, persistence, caching
├─────────────────────────────────────┤
│          Models / Schemas           │  ← Data structures: ORM models, validation schemas
├─────────────────────────────────────┤
│          Database / External APIs   │  ← Infrastructure: DB, LLM APIs, message queues
└─────────────────────────────────────┘
```

### Rules

1. Each layer only calls the layer directly below it
2. Route handlers are thin — validate input, call service, return response
3. Services contain all business logic — never in routes or repositories
4. Repositories handle only data access — no business rules
5. Models define data shape — no behavior beyond validation

---

## LLM Provider Abstraction

### Python

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass

@dataclass
class LLMResponse:
    content: str
    model: str
    usage: dict  # {"prompt_tokens": int, "completion_tokens": int, "total_tokens": int}
    finish_reason: str

@dataclass
class LLMMessage:
    role: str  # "system" | "user" | "assistant"
    content: str

class LLMProvider(ABC):
    @abstractmethod
    async def chat(self, messages: list[LLMMessage], **kwargs) -> LLMResponse: ...

    @abstractmethod
    async def chat_stream(self, messages: list[LLMMessage], **kwargs):
        """Yields content chunks as strings."""
        ...

class OpenAIProvider(LLMProvider):
    def __init__(self, client, model: str = "gpt-4o"):
        self.client = client
        self.model = model

    async def chat(self, messages: list[LLMMessage], **kwargs) -> LLMResponse:
        response = await self.client.chat.completions.create(
            model=self.model,
            messages=[{"role": m.role, "content": m.content} for m in messages],
            **kwargs,
        )
        choice = response.choices[0]
        return LLMResponse(
            content=choice.message.content or "",
            model=response.model,
            usage={
                "prompt_tokens": response.usage.prompt_tokens,
                "completion_tokens": response.usage.completion_tokens,
                "total_tokens": response.usage.total_tokens,
            },
            finish_reason=choice.finish_reason,
        )

    async def chat_stream(self, messages: list[LLMMessage], **kwargs):
        stream = await self.client.chat.completions.create(
            model=self.model,
            messages=[{"role": m.role, "content": m.content} for m in messages],
            stream=True,
            **kwargs,
        )
        async for chunk in stream:
            if chunk.choices[0].delta.content:
                yield chunk.choices[0].delta.content

class AnthropicProvider(LLMProvider):
    def __init__(self, client, model: str = "claude-sonnet-4-20250514"):
        self.client = client
        self.model = model

    async def chat(self, messages: list[LLMMessage], **kwargs) -> LLMResponse:
        system_msg = next((m.content for m in messages if m.role == "system"), None)
        chat_messages = [{"role": m.role, "content": m.content} for m in messages if m.role != "system"]
        response = await self.client.messages.create(
            model=self.model,
            max_tokens=kwargs.pop("max_tokens", 4096),
            system=system_msg or "",
            messages=chat_messages,
            **kwargs,
        )
        return LLMResponse(
            content=response.content[0].text,
            model=response.model,
            usage={
                "prompt_tokens": response.usage.input_tokens,
                "completion_tokens": response.usage.output_tokens,
                "total_tokens": response.usage.input_tokens + response.usage.output_tokens,
            },
            finish_reason=response.stop_reason,
        )

    async def chat_stream(self, messages: list[LLMMessage], **kwargs):
        system_msg = next((m.content for m in messages if m.role == "system"), None)
        chat_messages = [{"role": m.role, "content": m.content} for m in messages if m.role != "system"]
        async with self.client.messages.stream(
            model=self.model,
            max_tokens=kwargs.pop("max_tokens", 4096),
            system=system_msg or "",
            messages=chat_messages,
            **kwargs,
        ) as stream:
            async for text in stream.text_stream:
                yield text

class OpenRouterProvider(LLMProvider):
    """OpenRouter uses OpenAI-compatible API with a different base URL."""
    def __init__(self, client, model: str = "anthropic/claude-sonnet-4"):
        # client is an OpenAI instance with base_url="https://openrouter.ai/api/v1"
        self.client = client
        self.model = model

    async def chat(self, messages: list[LLMMessage], **kwargs) -> LLMResponse:
        response = await self.client.chat.completions.create(
            model=self.model,
            messages=[{"role": m.role, "content": m.content} for m in messages],
            **kwargs,
        )
        choice = response.choices[0]
        return LLMResponse(
            content=choice.message.content or "",
            model=response.model,
            usage={
                "prompt_tokens": response.usage.prompt_tokens,
                "completion_tokens": response.usage.completion_tokens,
                "total_tokens": response.usage.total_tokens,
            },
            finish_reason=choice.finish_reason,
        )

    async def chat_stream(self, messages: list[LLMMessage], **kwargs):
        stream = await self.client.chat.completions.create(
            model=self.model,
            messages=[{"role": m.role, "content": m.content} for m in messages],
            stream=True,
            **kwargs,
        )
        async for chunk in stream:
            if chunk.choices[0].delta.content:
                yield chunk.choices[0].delta.content

def create_llm_provider(settings) -> LLMProvider:
    """Factory that creates the configured LLM provider."""
    if settings.llm_provider == "openai":
        from openai import AsyncOpenAI
        client = AsyncOpenAI(api_key=settings.openai_api_key)
        return OpenAIProvider(client, model=settings.llm_model or "gpt-4o")
    elif settings.llm_provider == "anthropic":
        from anthropic import AsyncAnthropic
        client = AsyncAnthropic(api_key=settings.anthropic_api_key)
        return AnthropicProvider(client, model=settings.llm_model or "claude-sonnet-4-20250514")
    elif settings.llm_provider == "openrouter":
        from openai import AsyncOpenAI
        client = AsyncOpenAI(
            api_key=settings.openrouter_api_key,
            base_url="https://openrouter.ai/api/v1",
        )
        return OpenRouterProvider(client, model=settings.llm_model or "anthropic/claude-sonnet-4")
    raise ValueError(f"Unknown LLM provider: {settings.llm_provider}")
```

### TypeScript

```typescript
export interface LLMResponse {
  content: string;
  model: string;
  usage: { promptTokens: number; completionTokens: number; totalTokens: number };
  finishReason: string;
}

export interface LLMMessage {
  role: "system" | "user" | "assistant";
  content: string;
}

export interface LLMProvider {
  chat(messages: LLMMessage[], options?: Record<string, unknown>): Promise<LLMResponse>;
  chatStream(messages: LLMMessage[], options?: Record<string, unknown>): AsyncIterable<string>;
}

// Implementations follow the same pattern as Python above
// See the stack-specific reference files for full TypeScript implementations
```

---

## Repository Pattern

### Python Generic Repository

```python
from typing import TypeVar, Generic, Type
from sqlalchemy import select, func
from sqlalchemy.ext.asyncio import AsyncSession

T = TypeVar("T")

class BaseRepository(Generic[T]):
    def __init__(self, model: Type[T]):
        self.model = model

    async def find_by_id(self, session: AsyncSession, id: int | str) -> T | None:
        return await session.get(self.model, id)

    async def find_all(self, session: AsyncSession, skip: int = 0, limit: int = 100) -> list[T]:
        stmt = select(self.model).offset(skip).limit(limit)
        result = await session.execute(stmt)
        return list(result.scalars().all())

    async def create(self, session: AsyncSession, **kwargs) -> T:
        instance = self.model(**kwargs)
        session.add(instance)
        await session.flush()
        return instance

    async def update(self, session: AsyncSession, id: int | str, **kwargs) -> T | None:
        instance = await self.find_by_id(session, id)
        if not instance:
            return None
        for key, value in kwargs.items():
            setattr(instance, key, value)
        await session.flush()
        return instance

    async def delete(self, session: AsyncSession, id: int | str) -> bool:
        instance = await self.find_by_id(session, id)
        if not instance:
            return False
        await session.delete(instance)
        await session.flush()
        return True

    async def count(self, session: AsyncSession) -> int:
        stmt = select(func.count()).select_from(self.model)
        result = await session.execute(stmt)
        return result.scalar_one()
```

### TypeScript Generic Repository

```typescript
export interface BaseRepository<T> {
  findById(id: string): Promise<T | null>;
  findAll(options?: { skip?: number; limit?: number }): Promise<T[]>;
  create(data: Partial<T>): Promise<T>;
  update(id: string, data: Partial<T>): Promise<T | null>;
  delete(id: string): Promise<boolean>;
  count(): Promise<number>;
}
```

---

## Error Handling

### Custom Exceptions

```python
# Python
class AppError(Exception):
    def __init__(self, message: str, status_code: int = 500, error_code: str = "INTERNAL_ERROR"):
        self.message = message
        self.status_code = status_code
        self.error_code = error_code

class NotFoundError(AppError):
    def __init__(self, resource: str, id: str | int):
        super().__init__(f"{resource} with id {id} not found", 404, "NOT_FOUND")

class UnauthorizedError(AppError):
    def __init__(self, message: str = "Not authenticated"):
        super().__init__(message, 401, "UNAUTHORIZED")

class ForbiddenError(AppError):
    def __init__(self, message: str = "Permission denied"):
        super().__init__(message, 403, "FORBIDDEN")

class ValidationError(AppError):
    def __init__(self, message: str):
        super().__init__(message, 422, "VALIDATION_ERROR")

class ConflictError(AppError):
    def __init__(self, message: str):
        super().__init__(message, 409, "CONFLICT")
```

### Global Error Handler

```python
# Python (FastAPI)
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

def register_exception_handlers(app: FastAPI):
    @app.exception_handler(AppError)
    async def app_error_handler(request: Request, exc: AppError):
        return JSONResponse(
            status_code=exc.status_code,
            content={
                "error": {
                    "code": exc.error_code,
                    "message": exc.message,
                    "request_id": request.state.request_id if hasattr(request.state, "request_id") else None,
                }
            },
        )

    @app.exception_handler(Exception)
    async def unhandled_error_handler(request: Request, exc: Exception):
        logger.exception("Unhandled error", request_id=getattr(request.state, "request_id", None))
        return JSONResponse(
            status_code=500,
            content={"error": {"code": "INTERNAL_ERROR", "message": "An unexpected error occurred"}},
        )
```

### Consistent Error Response Format

```json
{
  "error": {
    "code": "NOT_FOUND",
    "message": "Document with id abc123 not found",
    "request_id": "req_7f3a8b2c"
  }
}
```

---

## Logging & Observability

### Python (structlog)

```python
import structlog
import uuid
from starlette.middleware.base import BaseHTTPMiddleware

structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer(),
    ],
)

logger = structlog.get_logger()

class LoggingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        request_id = str(uuid.uuid4())[:8]
        request.state.request_id = request_id

        structlog.contextvars.bind_contextvars(request_id=request_id)

        logger.info(
            "request_started",
            method=request.method,
            path=request.url.path,
        )

        response = await call_next(request)

        logger.info(
            "request_completed",
            method=request.method,
            path=request.url.path,
            status_code=response.status_code,
        )

        response.headers["X-Request-ID"] = request_id
        structlog.contextvars.unbind_contextvars("request_id")
        return response
```

### TypeScript (pino via Fastify)

Fastify has built-in pino logging. Configure via app factory:

```typescript
const app = Fastify({
  logger: {
    level: config.logLevel,
    serializers: {
      req(request) {
        return { method: request.method, url: request.url, id: request.id };
      },
    },
    transport: config.isDev ? { target: "pino-pretty" } : undefined,
  },
  genReqId: () => randomUUID().slice(0, 8),
});
```

---

## Configuration Management

### Python (pydantic-settings)

```python
from pydantic_settings import BaseSettings
from pydantic import Field

class Settings(BaseSettings):
    # App
    app_name: str = "ai-backend"
    app_version: str = "0.1.0"
    app_env: str = "development"
    debug: bool = False
    app_port: int = 8000

    # Database
    database_url: str  # Required, no default — fails fast if missing

    # Auth
    jwt_secret_key: str
    jwt_algorithm: str = "HS256"
    jwt_access_token_expire_minutes: int = 30
    jwt_refresh_token_expire_days: int = 7

    # LLM
    llm_provider: str = "openai"
    llm_model: str | None = None
    openai_api_key: str = ""
    anthropic_api_key: str = ""
    openrouter_api_key: str = ""

    # Embedding
    embedding_model: str = "text-embedding-3-small"
    embedding_dimensions: int = 1536

    # CORS
    cors_origins: list[str] = ["http://localhost:3000"]

    # Logging
    log_level: str = "INFO"

    model_config = {"env_file": ".env", "env_file_encoding": "utf-8"}

settings = Settings()
```

### TypeScript (Zod)

```typescript
import { z } from "zod";
import "dotenv/config";

const configSchema = z.object({
  appName: z.string().default("ai-backend"),
  port: z.coerce.number().default(3000),
  nodeEnv: z.enum(["development", "production", "test"]).default("development"),
  isDev: z.boolean().optional(),
  logLevel: z.string().default("info"),

  databaseUrl: z.string().min(1, "DATABASE_URL is required"),

  jwtSecret: z.string().min(32, "JWT_SECRET must be at least 32 characters"),
  jwtExpiresIn: z.string().default("30m"),
  jwtRefreshExpiresIn: z.string().default("7d"),

  llmProvider: z.enum(["openai", "anthropic", "openrouter"]).default("openai"),
  llmModel: z.string().optional(),
  openaiApiKey: z.string().default(""),
  anthropicApiKey: z.string().default(""),
  openrouterApiKey: z.string().default(""),

  embeddingModel: z.string().default("text-embedding-3-small"),
  embeddingDimensions: z.coerce.number().default(1536),

  corsOrigins: z.string().transform((s) => s.split(",")).default("http://localhost:3000"),
});

const parsed = configSchema.parse({
  appName: process.env.APP_NAME,
  port: process.env.APP_PORT,
  nodeEnv: process.env.NODE_ENV,
  logLevel: process.env.LOG_LEVEL,
  databaseUrl: process.env.DATABASE_URL ?? process.env.MONGODB_URL,
  jwtSecret: process.env.JWT_SECRET,
  jwtExpiresIn: process.env.JWT_EXPIRES_IN,
  jwtRefreshExpiresIn: process.env.JWT_REFRESH_EXPIRES_IN,
  llmProvider: process.env.LLM_PROVIDER,
  llmModel: process.env.LLM_MODEL,
  openaiApiKey: process.env.OPENAI_API_KEY,
  anthropicApiKey: process.env.ANTHROPIC_API_KEY,
  openrouterApiKey: process.env.OPENROUTER_API_KEY,
  embeddingModel: process.env.EMBEDDING_MODEL,
  embeddingDimensions: process.env.EMBEDDING_DIMENSIONS,
  corsOrigins: process.env.CORS_ORIGINS,
});

export const config = { ...parsed, isDev: parsed.nodeEnv === "development" };
export type Config = typeof config;
```

---

## Health Checks

### Python

```python
from fastapi import APIRouter, Response
from app.db.session import engine

health_router = APIRouter(tags=["health"])

@health_router.get("/health")
async def health_check():
    checks = {}

    # Database
    try:
        async with engine.connect() as conn:
            await conn.execute(text("SELECT 1"))
        checks["database"] = "healthy"
    except Exception as e:
        checks["database"] = f"unhealthy: {e}"

    # LLM Provider (lightweight check)
    checks["llm_provider"] = settings.llm_provider

    status = "healthy" if all(v == "healthy" or v == settings.llm_provider for v in checks.values()) else "unhealthy"
    status_code = 200 if status == "healthy" else 503

    return Response(
        content=json.dumps({"status": status, "checks": checks}),
        status_code=status_code,
        media_type="application/json",
    )
```

### TypeScript

```typescript
import { FastifyInstance } from "fastify";

export async function healthRoutes(app: FastifyInstance) {
  app.get("/health", async (request, reply) => {
    const checks: Record<string, string> = {};

    try {
      await app.prisma.$queryRaw`SELECT 1`; // or mongoose.connection.readyState
      checks.database = "healthy";
    } catch (err) {
      checks.database = `unhealthy: ${(err as Error).message}`;
    }

    const isHealthy = Object.values(checks).every((v) => v === "healthy");
    return reply.status(isHealthy ? 200 : 503).send({
      status: isHealthy ? "healthy" : "unhealthy",
      checks,
    });
  });
}
```

---

## Docker & Infrastructure

### Python Dockerfile

```dockerfile
FROM python:3.12-slim AS base

WORKDIR /app
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

# Install system deps
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Install Python deps
COPY pyproject.toml ./
RUN pip install --no-cache-dir -e ".[dev]" 2>/dev/null || pip install --no-cache-dir .

COPY . .

EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### TypeScript Dockerfile

```dockerfile
FROM node:22-slim AS base

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci

COPY . .

# Generate Prisma client if using Prisma
RUN npx prisma generate 2>/dev/null || true

EXPOSE 3000
CMD ["npx", "tsx", "src/index.ts"]
```

### Production Multi-stage (TypeScript)

```dockerfile
FROM node:22-slim AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npx tsc
RUN npx prisma generate 2>/dev/null || true

FROM node:22-slim AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./
COPY --from=builder /app/prisma ./prisma
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

---

## Testing Strategy

### Test Hierarchy

1. **Unit tests** — Services and utilities with mocked repositories
2. **Integration tests** — Repositories against a test database
3. **API tests** — HTTP endpoints end-to-end

### Python Test Setup

```python
import pytest
import pytest_asyncio
from httpx import AsyncClient, ASGITransport
from app.main import create_app

@pytest_asyncio.fixture
async def app():
    app = create_app()
    yield app

@pytest_asyncio.fixture
async def client(app):
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        yield client

@pytest.mark.asyncio
async def test_health(client):
    response = await client.get("/health")
    assert response.status_code == 200
```

### TypeScript Test Setup

```typescript
import { describe, it, expect, beforeAll, afterAll } from "vitest";
import { buildApp } from "../src/app";

let app: Awaited<ReturnType<typeof buildApp>>;

beforeAll(async () => {
  app = await buildApp();
});

afterAll(async () => {
  await app.close();
});

describe("Health", () => {
  it("returns 200", async () => {
    const res = await app.inject({ method: "GET", url: "/health" });
    expect(res.statusCode).toBe(200);
  });
});
```

---

## API Design

### Consistent Response Envelope

Success:
```json
{
  "data": { ... },
  "meta": {
    "request_id": "req_7f3a8b2c"
  }
}
```

List:
```json
{
  "data": [ ... ],
  "meta": {
    "total": 42,
    "page": 1,
    "per_page": 20,
    "request_id": "req_7f3a8b2c"
  }
}
```

Error:
```json
{
  "error": {
    "code": "NOT_FOUND",
    "message": "Document with id abc123 not found",
    "request_id": "req_7f3a8b2c"
  }
}
```

### Standard Endpoints for a Resource

```
GET    /api/v1/{resources}          # List (paginated, filterable)
POST   /api/v1/{resources}          # Create
GET    /api/v1/{resources}/:id      # Get by ID
PUT    /api/v1/{resources}/:id      # Full update
PATCH  /api/v1/{resources}/:id      # Partial update
DELETE /api/v1/{resources}/:id      # Delete
```

### Pagination Query Params

```
GET /api/v1/documents?page=1&per_page=20&sort=-created_at&filter[status]=active
```
