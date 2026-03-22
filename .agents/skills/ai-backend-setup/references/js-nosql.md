# JavaScript/TypeScript + MongoDB + Atlas Vector Search Stack Reference

## Table of Contents
1. [Dependencies](#dependencies)
2. [Project Structure](#project-structure)
3. [Database Setup](#database-setup)
4. [ODM Configuration](#odm-configuration)
5. [Vector Search Setup](#vector-search-setup)
6. [Search Implementations](#search-implementations)
7. [Fastify Application Factory](#fastify-application-factory)

---

## Dependencies

### Core (package.json) — Mongoose variant

```json
{
  "dependencies": {
    "fastify": "^5.2.0",
    "@fastify/cors": "^11.0.0",
    "@fastify/helmet": "^13.0.0",
    "@fastify/rate-limit": "^10.2.0",
    "@fastify/jwt": "^9.0.0",
    "@fastify/sensible": "^6.0.0",
    "mongoose": "^8.9.0",
    "zod": "^3.24.0",
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
    "@typescript-eslint/eslint-plugin": "^8.18.0"
  }
}
```

### Prisma MongoDB variant (replace mongoose)
```json
{
  "dependencies": {
    "@prisma/client": "^6.2.0"
  },
  "devDependencies": {
    "prisma": "^6.2.0"
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
│   │   ├── connection.ts           # Mongoose/Prisma connection
│   │   ├── indexes.ts              # Atlas Search + Vector index setup
│   │   └── seed.ts                 # Initial admin + roles seed
│   │
│   ├── models/
│   │   ├── user.model.ts           # User schema + model
│   │   ├── api-key.model.ts        # API key schema + model
│   │   ├── permission.model.ts     # Permission schema + model
│   │   └── document.model.ts       # Domain model
│   │
│   ├── schemas/
│   │   ├── auth.schema.ts          # Zod schemas for auth
│   │   ├── user.schema.ts          # Zod schemas for users
│   │   └── document.schema.ts      # Zod schemas for domain
│   │
│   ├── repositories/
│   │   ├── base.repository.ts      # Generic repository interface
│   │   ├── user.repository.ts
│   │   ├── api-key.repository.ts
│   │   └── document.repository.ts
│   │
│   ├── services/
│   │   ├── auth.service.ts
│   │   ├── user.service.ts
│   │   ├── llm.service.ts
│   │   ├── embedding.service.ts
│   │   ├── search.service.ts
│   │   └── document.service.ts
│   │
│   ├── routes/
│   │   ├── v1/
│   │   │   ├── index.ts
│   │   │   ├── auth.routes.ts
│   │   │   ├── user.routes.ts
│   │   │   └── document.routes.ts
│   │   └── health.routes.ts
│   │
│   ├── plugins/
│   │   ├── auth.plugin.ts
│   │   ├── error-handler.plugin.ts
│   │   └── mongoose.plugin.ts
│   │
│   ├── middleware/
│   │   ├── auth.middleware.ts
│   │   └── validate.middleware.ts
│   │
│   ├── core/
│   │   ├── security.ts
│   │   ├── permissions.ts
│   │   ├── errors.ts
│   │   └── constants.ts
│   │
│   └── llm/
│       ├── base.provider.ts
│       ├── openai.provider.ts
│       ├── anthropic.provider.ts
│       └── openrouter.provider.ts
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

### Mongoose Connection (db/connection.ts)

```typescript
import mongoose from "mongoose";
import { config } from "../config";

export async function connectDatabase(): Promise<void> {
  mongoose.set("strictQuery", true);

  await mongoose.connect(config.mongodbUrl, {
    maxPoolSize: 20,
    minPoolSize: 5,
    serverSelectionTimeoutMS: 5000,
    socketTimeoutMS: 45000,
  });

  mongoose.connection.on("error", (err) => {
    console.error("MongoDB connection error:", err);
  });
}

export async function disconnectDatabase(): Promise<void> {
  await mongoose.disconnect();
}

export function getDatabase() {
  return mongoose.connection.db;
}
```

---

## ODM Configuration

### Mongoose Schemas

```typescript
import mongoose, { Schema, Document as MongooseDoc, Types } from "mongoose";

// ---- User ----
export interface IUser extends MongooseDoc {
  email: string;
  passwordHash: string;
  name: string;
  role: "ADMIN" | "USER";
  isActive: boolean;
  createdAt: Date;
  updatedAt: Date;
}

const userSchema = new Schema<IUser>(
  {
    email: { type: String, required: true, unique: true, lowercase: true, trim: true },
    passwordHash: { type: String, required: true, select: false },
    name: { type: String, required: true, trim: true },
    role: { type: String, enum: ["ADMIN", "USER"], default: "USER" },
    isActive: { type: Boolean, default: true },
  },
  { timestamps: true },
);

userSchema.index({ email: 1 });
userSchema.index({ role: 1 });

export const User = mongoose.model<IUser>("User", userSchema);

// ---- API Key ----
export interface IApiKey extends MongooseDoc {
  key: string;
  name: string;
  userId: Types.ObjectId;
  expiresAt?: Date;
  isActive: boolean;
}

const apiKeySchema = new Schema<IApiKey>(
  {
    key: { type: String, required: true, unique: true },
    name: { type: String, required: true },
    userId: { type: Schema.Types.ObjectId, ref: "User", required: true },
    expiresAt: { type: Date },
    isActive: { type: Boolean, default: true },
  },
  { timestamps: true },
);

apiKeySchema.index({ key: 1 });
apiKeySchema.index({ userId: 1 });

export const ApiKey = mongoose.model<IApiKey>("ApiKey", apiKeySchema);

// ---- Permission ----
export interface IPermission extends MongooseDoc {
  role: "ADMIN" | "USER";
  resource: string;
  action: string;
}

const permissionSchema = new Schema<IPermission>({
  role: { type: String, enum: ["ADMIN", "USER"], required: true },
  resource: { type: String, required: true },
  action: { type: String, required: true },
});

permissionSchema.index({ role: 1, resource: 1, action: 1 }, { unique: true });

export const Permission = mongoose.model<IPermission>("Permission", permissionSchema);

// ---- Document (domain model) ----
export interface IDocument extends MongooseDoc {
  title: string;
  content: string;
  embedding?: number[];
  metadata?: Record<string, unknown>;
  ownerId: Types.ObjectId;
  createdAt: Date;
  updatedAt: Date;
}

const documentSchema = new Schema<IDocument>(
  {
    title: { type: String, required: true },
    content: { type: String, required: true },
    embedding: { type: [Number], select: false },
    metadata: { type: Schema.Types.Mixed },
    ownerId: { type: Schema.Types.ObjectId, ref: "User", required: true },
  },
  { timestamps: true },
);

documentSchema.index({ ownerId: 1 });
documentSchema.index({ title: "text", content: "text" });

export const DocumentModel = mongoose.model<IDocument>("Document", documentSchema);
```

### Prisma MongoDB Schema Alternative

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "mongodb"
  url      = env("MONGODB_URL")
}

enum Role {
  ADMIN
  USER
}

model User {
  id           String    @id @default(auto()) @map("_id") @db.ObjectId
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
  id        String    @id @default(auto()) @map("_id") @db.ObjectId
  key       String    @unique
  name      String
  userId    String    @map("user_id") @db.ObjectId
  user      User      @relation(fields: [userId], references: [id], onDelete: Cascade)
  expiresAt DateTime? @map("expires_at")
  isActive  Boolean   @default(true) @map("is_active")
  createdAt DateTime  @default(now()) @map("created_at")

  @@map("api_keys")
}

model Document {
  id        String   @id @default(auto()) @map("_id") @db.ObjectId
  title     String
  content   String
  metadata  Json?
  ownerId   String   @map("owner_id") @db.ObjectId
  owner     User     @relation(fields: [ownerId], references: [id])
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")

  @@map("documents")
}
```

**Note:** Prisma MongoDB does not support custom vector fields natively. For vector operations, use raw MongoDB aggregation via `prisma.$runCommandRaw()`.

---

## Vector Search Setup

### Atlas Vector Search Index Creation (db/indexes.ts)

```typescript
import { Db } from "mongodb";

export async function createVectorSearchIndex(db: Db): Promise<void> {
  const collection = db.collection("documents");

  try {
    await collection.createSearchIndex({
      name: "vector_index",
      type: "vectorSearch",
      definition: {
        fields: [
          {
            type: "vector",
            path: "embedding",
            numDimensions: 1536,
            similarity: "cosine",
          },
          { type: "filter", path: "ownerId" },
        ],
      },
    });
  } catch (err: any) {
    if (!err.message?.includes("already exists")) throw err;
  }
}

export async function createAtlasSearchIndex(db: Db): Promise<void> {
  const collection = db.collection("documents");

  try {
    await collection.createSearchIndex({
      name: "search_index",
      type: "search",
      definition: {
        mappings: {
          dynamic: false,
          fields: {
            title: { type: "string", analyzer: "lucene.standard" },
            content: { type: "string", analyzer: "lucene.standard" },
          },
        },
      },
    });
  } catch (err: any) {
    if (!err.message?.includes("already exists")) throw err;
  }
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

### Vectorless (Atlas Full-Text Search)

```typescript
import { Collection, Document as MongoDoc } from "mongodb";

export async function fulltextSearch(
  collection: Collection,
  query: string,
  limit = 10,
  ownerId?: string,
): Promise<MongoDoc[]> {
  const pipeline: MongoDoc[] = [
    {
      $search: {
        index: "search_index",
        text: {
          query,
          path: ["title", "content"],
        },
      },
    },
    { $addFields: { searchScore: { $meta: "searchScore" } } },
  ];

  if (ownerId) {
    pipeline.push({ $match: { ownerId } });
  }
  pipeline.push({ $limit: limit });

  return collection.aggregate(pipeline).toArray();
}
```

### Vector Only

```typescript
export async function vectorSearch(
  collection: Collection,
  queryEmbedding: number[],
  limit = 10,
  ownerId?: string,
): Promise<MongoDoc[]> {
  const vectorStage: MongoDoc = {
    $vectorSearch: {
      index: "vector_index",
      path: "embedding",
      queryVector: queryEmbedding,
      numCandidates: limit * 10,
      limit,
    },
  };

  if (ownerId) {
    vectorStage.$vectorSearch.filter = { ownerId };
  }

  const pipeline: MongoDoc[] = [
    vectorStage,
    { $addFields: { vectorScore: { $meta: "vectorSearchScore" } } },
  ];

  return collection.aggregate(pipeline).toArray();
}
```

### Hybrid (RRF Fusion)

```typescript
interface ScoredDoc {
  _id: string;
  score: number;
  [key: string]: unknown;
}

export async function hybridSearch(
  collection: Collection,
  query: string,
  queryEmbedding: number[],
  limit = 10,
  ownerId?: string,
): Promise<ScoredDoc[]> {
  const fetchLimit = limit * 2;
  const rrfK = 60;

  const [ftsResults, vecResults] = await Promise.all([
    fulltextSearch(collection, query, fetchLimit, ownerId),
    vectorSearch(collection, queryEmbedding, fetchLimit, ownerId),
  ]);

  const scores = new Map<string, number>();
  const docMap = new Map<string, MongoDoc>();

  ftsResults.forEach((doc, rank) => {
    const id = doc._id.toString();
    scores.set(id, (scores.get(id) ?? 0) + 0.5 / (rrfK + rank + 1));
    docMap.set(id, doc);
  });

  vecResults.forEach((doc, rank) => {
    const id = doc._id.toString();
    scores.set(id, (scores.get(id) ?? 0) + 0.5 / (rrfK + rank + 1));
    docMap.set(id, doc);
  });

  return [...scores.entries()]
    .sort((a, b) => b[1] - a[1])
    .slice(0, limit)
    .map(([id, score]) => ({ ...docMap.get(id)!, _id: id, score }));
}
```

#### Native $rankFusion (Atlas 8.0+)

```typescript
export async function hybridSearchNative(
  collection: Collection,
  query: string,
  queryEmbedding: number[],
  limit = 10,
): Promise<MongoDoc[]> {
  const pipeline: MongoDoc[] = [
    {
      $rankFusion: {
        input: {
          pipelines: {
            vector: [
              {
                $vectorSearch: {
                  index: "vector_index",
                  path: "embedding",
                  queryVector: queryEmbedding,
                  numCandidates: limit * 10,
                  limit: limit * 2,
                },
              },
            ],
            text: [
              {
                $search: {
                  index: "search_index",
                  text: { query, path: ["title", "content"] },
                },
              },
              { $limit: limit * 2 },
            ],
          },
        },
      },
    },
    { $limit: limit },
  ];

  return collection.aggregate(pipeline).toArray();
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
import { mongoosePlugin } from "./plugins/mongoose.plugin";
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

  await app.register(helmet);
  await app.register(cors, {
    origin: config.corsOrigins,
    credentials: true,
  });
  await app.register(rateLimit, { max: 100, timeWindow: "1 minute" });
  await app.register(sensible);

  await app.register(errorHandlerPlugin);
  await app.register(mongoosePlugin);
  await app.register(authPlugin);

  await app.register(healthRoutes);
  await app.register(v1Routes, { prefix: "/api/v1" });

  return app;
}
```

### Mongoose Plugin (plugins/mongoose.plugin.ts)

```typescript
import fp from "fastify-plugin";
import { FastifyInstance } from "fastify";
import { connectDatabase, disconnectDatabase } from "../db/connection";

export const mongoosePlugin = fp(async (app: FastifyInstance) => {
  await connectDatabase();
  app.log.info("MongoDB connected");

  app.addHook("onClose", async () => {
    await disconnectDatabase();
    app.log.info("MongoDB disconnected");
  });
});
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
      mongodb:
        condition: service_healthy
    volumes:
      - .:/app
      - /app/node_modules
    command: npx tsx watch src/index.ts

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

**Note:** `mongodb/mongodb-atlas-local` supports Atlas Search and Vector Search for local development. Production should use MongoDB Atlas (cloud).

## .env.example

```bash
# Application
APP_NAME=ai-backend
APP_PORT=3000
NODE_ENV=development
LOG_LEVEL=info

# MongoDB
MONGODB_URL=mongodb://admin:admin@mongodb:27017/ai_backend?authSource=admin
DB_NAME=ai_backend
MONGO_USER=admin
MONGO_PASSWORD=admin
MONGO_PORT=27017

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
