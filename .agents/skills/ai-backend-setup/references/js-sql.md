# JavaScript/TypeScript + PostgreSQL + pgvector Stack Reference

## Table of Contents
1. [Dependencies](#dependencies)
2. [Project Structure](#project-structure)
3. [Database Setup](#database-setup)
4. [ORM Configuration](#orm-configuration)
5. [Vector Search Setup](#vector-search-setup)
6. [Search Implementations](#search-implementations)
7. [Fastify Application Factory](#fastify-application-factory)

---

## Dependencies

### Core (package.json)

```json
{
  "dependencies": {
    "fastify": "^5.2.0",
    "@fastify/cors": "^11.0.0",
    "@fastify/helmet": "^13.0.0",
    "@fastify/rate-limit": "^10.2.0",
    "@fastify/jwt": "^9.0.0",
    "@fastify/sensible": "^6.0.0",
    "zod": "^3.24.0",
    "pgvector": "^0.2.0",
    "bcryptjs": "^2.4.3",
    "pino": "^9.6.0",
    "pino-pretty": "^13.0.0",
    "dotenv": "^16.4.0",
    "openai": "^4.80.0",
    "@anthropic-ai/sdk": "^0.36.0",
    "uuid": "^11.0.0"
  },
  "devDependencies": {
    "typescript": "^5.7.0",
    "tsx": "^4.19.0",
    "@types/node": "^22.0.0",
    "@types/bcryptjs": "^2.4.0",
    "vitest": "^3.0.0",
    "eslint": "^9.17.0",
    "@typescript-eslint/eslint-plugin": "^8.18.0",
    "prisma": "^6.2.0"
  }
}
```

### Prisma ORM Addition
```json
{
  "dependencies": {
    "@prisma/client": "^6.2.0"
  }
}
```

### Drizzle ORM Alternative (replace Prisma)
```json
{
  "dependencies": {
    "drizzle-orm": "^0.38.0",
    "postgres": "^3.4.0"
  },
  "devDependencies": {
    "drizzle-kit": "^0.30.0"
  }
}
```

---

## Project Structure

```
project-name/
├── src/
│   ├── index.ts                    # Entry point
│   ├── app.ts                      # Fastify app factory
│   ├── config/
│   │   └── index.ts                # Env config with Zod validation
│   │
│   ├── db/
│   │   ├── client.ts               # Prisma/Drizzle client
│   │   ├── migrations/             # Database migrations
│   │   └── seed.ts                 # Initial admin + roles seed
│   │
│   ├── models/
│   │   ├── user.ts                 # User types and Zod schemas
│   │   ├── api-key.ts              # API key types
│   │   └── document.ts             # Domain model types
│   │
│   ├── repositories/
│   │   ├── base.repository.ts      # Generic repository interface
│   │   ├── user.repository.ts
│   │   ├── api-key.repository.ts
│   │   └── document.repository.ts
│   │
│   ├── services/
│   │   ├── auth.service.ts         # JWT + API key auth
│   │   ├── user.service.ts
│   │   ├── llm.service.ts          # LLM provider abstraction
│   │   ├── embedding.service.ts    # Embedding generation
│   │   ├── search.service.ts       # Vector/hybrid/fulltext
│   │   └── document.service.ts     # Domain service
│   │
│   ├── routes/
│   │   ├── v1/
│   │   │   ├── index.ts            # Route aggregator
│   │   │   ├── auth.routes.ts
│   │   │   ├── user.routes.ts
│   │   │   └── document.routes.ts
│   │   └── health.routes.ts
│   │
│   ├── plugins/
│   │   ├── auth.plugin.ts          # JWT + API key auth plugin
│   │   ├── error-handler.plugin.ts # Global error handling
│   │   └── prisma.plugin.ts        # Prisma client lifecycle
│   │
│   ├── middleware/
│   │   ├── auth.middleware.ts       # Auth guard + RBAC
│   │   └── validate.middleware.ts   # Zod validation
│   │
│   ├── core/
│   │   ├── security.ts             # Password hash, JWT utils
│   │   ├── permissions.ts          # RBAC helpers
│   │   ├── errors.ts               # Custom error classes
│   │   └── constants.ts            # Roles, permissions enums
│   │
│   └── llm/
│       ├── base.provider.ts        # Abstract LLM interface
│       ├── openai.provider.ts
│       ├── anthropic.provider.ts
│       └── openrouter.provider.ts
│
├── prisma/
│   └── schema.prisma               # Prisma schema (if using Prisma)
│
├── tests/
│   ├── setup.ts
│   ├── auth.test.ts
│   └── document.test.ts
│
├── Dockerfile
├── docker-compose.yml
├── .env.example
├── package.json
├── tsconfig.json
└── README.md
```

---

## Database Setup

### Prisma Schema (prisma/schema.prisma)

```prisma
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["postgresqlExtensions"]
}

datasource db {
  provider   = "postgresql"
  url        = env("DATABASE_URL")
  extensions = [pgvector(map: "vector", schema: "public")]
}

enum Role {
  ADMIN
  USER
}

model User {
  id           String    @id @default(uuid())
  email        String    @unique
  passwordHash String    @map("password_hash")
  name         String
  role         Role      @default(USER)
  isActive     Boolean   @default(true) @map("is_active")
  createdAt    DateTime  @default(now()) @map("created_at")
  updatedAt    DateTime  @updatedAt @map("updated_at")
  apiKeys      ApiKey[]
  documents    Document[]

  @@map("users")
}

model ApiKey {
  id        String   @id @default(uuid())
  key       String   @unique
  name      String
  userId    String   @map("user_id")
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  expiresAt DateTime? @map("expires_at")
  isActive  Boolean  @default(true) @map("is_active")
  createdAt DateTime @default(now()) @map("created_at")

  @@map("api_keys")
}

model Permission {
  id       String @id @default(uuid())
  role     Role
  resource String
  action   String

  @@unique([role, resource, action])
  @@map("permissions")
}

model Document {
  id        String                   @id @default(uuid())
  title     String
  content   String
  embedding Unsupported("vector(1536)")?
  metadata  Json?
  ownerId   String                   @map("owner_id")
  owner     User                     @relation(fields: [ownerId], references: [id])
  createdAt DateTime                 @default(now()) @map("created_at")
  updatedAt DateTime                 @updatedAt @map("updated_at")

  @@map("documents")
}
```

### Drizzle Schema Alternative (src/db/schema.ts)

```typescript
import { pgTable, uuid, varchar, text, boolean, timestamp, json, index, pgEnum } from "drizzle-orm/pg-core";
import { customType } from "drizzle-orm/pg-core";

const vector = customType<{ data: number[]; driverParam: string }>({
  dataType() { return "vector(1536)"; },
  toDriver(value: number[]) { return `[${value.join(",")}]`; },
  fromDriver(value: string) { return JSON.parse(value); },
});

export const roleEnum = pgEnum("role", ["ADMIN", "USER"]);

export const users = pgTable("users", {
  id: uuid("id").primaryKey().defaultRandom(),
  email: varchar("email", { length: 255 }).unique().notNull(),
  passwordHash: varchar("password_hash", { length: 255 }).notNull(),
  name: varchar("name", { length: 255 }).notNull(),
  role: roleEnum("role").default("USER").notNull(),
  isActive: boolean("is_active").default(true).notNull(),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
});

export const documents = pgTable("documents", {
  id: uuid("id").primaryKey().defaultRandom(),
  title: varchar("title", { length: 500 }).notNull(),
  content: text("content").notNull(),
  embedding: vector("embedding"),
  metadata: json("metadata"),
  ownerId: uuid("owner_id").references(() => users.id).notNull(),
  createdAt: timestamp("created_at", { withTimezone: true }).defaultNow().notNull(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).defaultNow().notNull(),
}, (table) => [
  index("documents_owner_idx").on(table.ownerId),
]);
```

---

## Vector Search Setup

### pgvector Extension Setup

```typescript
import { PrismaClient } from "@prisma/client";

export async function initVectorExtension(prisma: PrismaClient) {
  await prisma.$executeRawUnsafe("CREATE EXTENSION IF NOT EXISTS vector");
}

export async function createVectorIndex(prisma: PrismaClient) {
  await prisma.$executeRawUnsafe(`
    CREATE INDEX IF NOT EXISTS documents_embedding_hnsw
    ON documents USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64)
  `);
}

export async function createFulltextIndex(prisma: PrismaClient) {
  await prisma.$executeRawUnsafe(`
    CREATE INDEX IF NOT EXISTS documents_content_fts
    ON documents USING gin(to_tsvector('english', content))
  `);
}
```

### Embedding Service

```typescript
import OpenAI from "openai";

export interface EmbeddingProvider {
  embed(text: string): Promise<number[]>;
  embedBatch(texts: string[]): Promise<number[][]>;
}

export class OpenAIEmbedding implements EmbeddingProvider {
  constructor(
    private client: OpenAI,
    private model = "text-embedding-3-small",
  ) {}

  async embed(text: string): Promise<number[]> {
    const response = await this.client.embeddings.create({
      input: text,
      model: this.model,
    });
    return response.data[0].embedding;
  }

  async embedBatch(texts: string[]): Promise<number[][]> {
    const response = await this.client.embeddings.create({
      input: texts,
      model: this.model,
    });
    return response.data.map((d) => d.embedding);
  }
}
```

---

## Search Implementations

### Vectorless (Full-Text Only)

```typescript
import { PrismaClient, Prisma } from "@prisma/client";

export async function fulltextSearch(prisma: PrismaClient, query: string, limit = 10) {
  return prisma.$queryRaw`
    SELECT id, title, content, metadata, owner_id, created_at,
           ts_rank(to_tsvector('english', content), plainto_tsquery('english', ${query})) AS rank
    FROM documents
    WHERE to_tsvector('english', content) @@ plainto_tsquery('english', ${query})
    ORDER BY rank DESC
    LIMIT ${limit}
  `;
}
```

### Vector Only

```typescript
export async function vectorSearch(
  prisma: PrismaClient,
  queryEmbedding: number[],
  limit = 10,
) {
  const embeddingStr = `[${queryEmbedding.join(",")}]`;
  return prisma.$queryRaw`
    SELECT id, title, content, metadata, owner_id, created_at,
           1 - (embedding <=> ${embeddingStr}::vector) AS similarity
    FROM documents
    WHERE embedding IS NOT NULL
    ORDER BY embedding <=> ${embeddingStr}::vector
    LIMIT ${limit}
  `;
}
```

### Hybrid (RRF Fusion)

```typescript
interface ScoredDoc {
  id: string;
  score: number;
  [key: string]: unknown;
}

export async function hybridSearch(
  prisma: PrismaClient,
  query: string,
  queryEmbedding: number[],
  limit = 10,
  rrfK = 60,
  semanticWeight = 0.5,
): Promise<ScoredDoc[]> {
  const fetchLimit = limit * 2;

  const [ftsResults, vecResults] = await Promise.all([
    fulltextSearch(prisma, query, fetchLimit) as Promise<Array<{ id: string } & Record<string, unknown>>>,
    vectorSearch(prisma, queryEmbedding, fetchLimit) as Promise<Array<{ id: string } & Record<string, unknown>>>,
  ]);

  const scores = new Map<string, number>();
  const docMap = new Map<string, Record<string, unknown>>();
  const textWeight = 1.0 - semanticWeight;

  ftsResults.forEach((doc, rank) => {
    scores.set(doc.id, (scores.get(doc.id) ?? 0) + textWeight / (rrfK + rank + 1));
    docMap.set(doc.id, doc);
  });

  vecResults.forEach((doc, rank) => {
    scores.set(doc.id, (scores.get(doc.id) ?? 0) + semanticWeight / (rrfK + rank + 1));
    docMap.set(doc.id, doc);
  });

  return [...scores.entries()]
    .sort((a, b) => b[1] - a[1])
    .slice(0, limit)
    .map(([id, score]) => ({ ...docMap.get(id)!, id, score }));
}
```

---

## Fastify Application Factory

```typescript
import Fastify, { FastifyInstance } from "fastify";
import cors from "@fastify/cors";
import helmet from "@fastify/helmet";
import rateLimit from "@fastify/rate-limit";
import sensible from "@fastify/sensible";
import { config } from "./config";
import { prismaPlugin } from "./plugins/prisma.plugin";
import { authPlugin } from "./plugins/auth.plugin";
import { errorHandlerPlugin } from "./plugins/error-handler.plugin";
import { healthRoutes } from "./routes/health.routes";
import { v1Routes } from "./routes/v1";

export async function buildApp(): Promise<FastifyInstance> {
  const app = Fastify({
    logger: {
      level: config.logLevel,
      transport: config.isDev ? { target: "pino-pretty" } : undefined,
    },
    requestId: true,
  });

  // Security
  await app.register(helmet);
  await app.register(cors, {
    origin: config.corsOrigins,
    credentials: true,
  });
  await app.register(rateLimit, {
    max: 100,
    timeWindow: "1 minute",
  });
  await app.register(sensible);

  // Plugins
  await app.register(errorHandlerPlugin);
  await app.register(prismaPlugin);
  await app.register(authPlugin);

  // Routes
  await app.register(healthRoutes);
  await app.register(v1Routes, { prefix: "/api/v1" });

  return app;
}
```

### Entry Point (src/index.ts)

```typescript
import { buildApp } from "./app";
import { config } from "./config";

async function main() {
  const app = await buildApp();

  try {
    await app.listen({ port: config.port, host: "0.0.0.0" });
  } catch (err) {
    app.log.error(err);
    process.exit(1);
  }
}

main();
```

---

## Docker Compose

```yaml
services:
  app:
    build: .
    ports:
      - "${APP_PORT:-3000}:3000"
    env_file: .env
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - .:/app
      - /app/node_modules
    command: npx tsx watch src/index.ts

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

## tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "tests"]
}
```

## .env.example

```bash
# Application
APP_NAME=ai-backend
APP_PORT=3000
NODE_ENV=development
LOG_LEVEL=info

# Database
DATABASE_URL=postgresql://postgres:postgres@db:5432/ai_backend
DB_USER=postgres
DB_PASSWORD=postgres
DB_NAME=ai_backend
DB_PORT=5432

# Auth
JWT_SECRET=CHANGE_ME_TO_A_RANDOM_SECRET
JWT_EXPIRES_IN=30m
JWT_REFRESH_EXPIRES_IN=7d

# LLM Provider
LLM_PROVIDER=openai
OPENAI_API_KEY=
ANTHROPIC_API_KEY=
OPENROUTER_API_KEY=

# Embedding
EMBEDDING_MODEL=text-embedding-3-small
EMBEDDING_DIMENSIONS=1536

# CORS
CORS_ORIGINS=http://localhost:5173,http://localhost:3000
```
