# Authentication & Authorization Patterns

## Table of Contents
1. [RBAC Model](#rbac-model)
2. [JWT Authentication](#jwt-authentication)
3. [Static API Key Authentication](#static-api-key-authentication)
4. [Dual Auth Strategy](#dual-auth-strategy)
5. [Python Implementation](#python-implementation)
6. [JavaScript Implementation](#javascript-implementation)
7. [Database Seed Script](#database-seed-script)

---

## RBAC Model

### Roles

| Role | Description |
|------|-------------|
| `ADMIN` | Full system access. Can manage users, roles, API keys, and all resources. |
| `USER` | Standard access. Can manage own resources and perform domain operations. |

### Permission Structure

Permissions follow the `resource:action` pattern:

| Permission | Admin | User |
|-----------|-------|------|
| `users:create` | Yes | No |
| `users:read` | All users | Own profile |
| `users:update` | All users | Own profile |
| `users:delete` | Yes | No |
| `api_keys:create` | Yes | Own keys |
| `api_keys:read` | All keys | Own keys |
| `api_keys:revoke` | All keys | Own keys |
| `documents:create` | Yes | Yes |
| `documents:read` | All | Own |
| `documents:update` | All | Own |
| `documents:delete` | All | Own |
| `search:query` | Yes | Yes |
| `llm:generate` | Yes | Yes |
| `admin:dashboard` | Yes | No |

### Permission Model Schema

```
Permission {
  id: string
  role: enum(ADMIN, USER)
  resource: string        # e.g., "users", "documents", "search"
  action: string          # e.g., "create", "read", "update", "delete"
}
```

Permissions are stored in the database and loaded at startup into a lookup structure for fast evaluation. This allows adding new permissions without code changes.

---

## JWT Authentication

### Token Pair Strategy

Use access + refresh token pattern:

- **Access Token**: Short-lived (15-30 minutes), contains user ID, role, and permissions. Sent in `Authorization: Bearer <token>` header.
- **Refresh Token**: Long-lived (7 days), contains only user ID. Used to obtain new access tokens. Stored as httpOnly cookie or in body.

### JWT Payload Structure

```json
{
  "sub": "user-uuid-here",
  "email": "user@example.com",
  "role": "ADMIN",
  "permissions": ["users:read", "users:create", "documents:read"],
  "type": "access",
  "iat": 1711000000,
  "exp": 1711001800
}
```

### Security Requirements

- Sign with HS256 (symmetric) for single-service, RS256 (asymmetric) for multi-service
- Secret key minimum 256 bits, loaded from environment variable
- Always validate `exp` claim
- Include `type` claim to prevent refresh tokens being used as access tokens
- Store refresh tokens hashed in database for revocation capability
- Implement token rotation: issuing a new refresh token invalidates the old one

### Auth Endpoints

```
POST /api/v1/auth/register       # Create account (returns tokens)
POST /api/v1/auth/login          # Email + password (returns tokens)
POST /api/v1/auth/refresh        # Refresh token (returns new access token)
POST /api/v1/auth/logout         # Invalidate refresh token
GET  /api/v1/auth/me             # Get current user profile
```

---

## Static API Key Authentication

### Use Cases

- Server-to-server communication
- CI/CD pipelines
- External integrations
- Automated scripts
- Webhook receivers

### API Key Format

Generate cryptographically secure keys with a recognizable prefix:

```
Format: {prefix}_{random_bytes_base64}
Example: aib_sk_a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6
```

- Prefix `aib_sk_` identifies it as an AI Backend secret key
- 32 bytes of random data, base62 encoded
- Store ONLY the hash in the database (SHA-256), never the raw key
- Show the raw key exactly once at creation time

### API Key Model

```
ApiKey {
  id: string
  keyHash: string          # SHA-256 of the actual key
  keyPrefix: string        # First 8 chars for identification (e.g., "aib_sk_a1")
  name: string             # Human-readable label
  userId: string           # Owner
  role: enum(ADMIN, USER)  # Inherits role or has explicit role
  expiresAt: datetime?     # Null = never expires
  lastUsedAt: datetime?
  isActive: boolean
  createdAt: datetime
}
```

### API Key Endpoints

```
POST   /api/v1/api-keys          # Create new key (returns raw key ONCE)
GET    /api/v1/api-keys          # List user's keys (prefix + metadata only)
DELETE /api/v1/api-keys/:id      # Revoke a key
```

### Authentication Flow

1. Client sends `Authorization: Bearer aib_sk_a1b2c3...` or `X-API-Key: aib_sk_a1b2c3...`
2. Server hashes the key with SHA-256
3. Look up hash in database
4. Verify `isActive` and `expiresAt`
5. Load associated user and role
6. Update `lastUsedAt` (debounced, not on every request)

---

## Dual Auth Strategy

The system supports BOTH JWT and API Key authentication. The auth middleware should:

1. Check for `Authorization: Bearer` header
2. If token starts with the API key prefix (`aib_sk_`), use API key auth flow
3. Otherwise, treat as JWT and validate accordingly
4. Both flows result in a unified `AuthContext` object attached to the request

### AuthContext

```
AuthContext {
  userId: string
  email: string
  role: "ADMIN" | "USER"
  permissions: string[]
  authMethod: "jwt" | "api_key"
  apiKeyId?: string        # Present only for API key auth
}
```

---

## Python Implementation

### Security Utilities (core/security.py)

```python
import hashlib
import secrets
from datetime import datetime, timedelta, timezone
from jose import jwt, JWTError
from passlib.context import CryptContext
from app.config import settings

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

API_KEY_PREFIX = "aib_sk_"

def hash_password(password: str) -> str:
    return pwd_context.hash(password)

def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)

def create_access_token(data: dict, expires_delta: timedelta | None = None) -> str:
    to_encode = data.copy()
    expire = datetime.now(timezone.utc) + (expires_delta or timedelta(minutes=settings.jwt_access_token_expire_minutes))
    to_encode.update({"exp": expire, "type": "access"})
    return jwt.encode(to_encode, settings.jwt_secret_key, algorithm=settings.jwt_algorithm)

def create_refresh_token(user_id: str) -> str:
    expire = datetime.now(timezone.utc) + timedelta(days=settings.jwt_refresh_token_expire_days)
    return jwt.encode(
        {"sub": user_id, "exp": expire, "type": "refresh"},
        settings.jwt_secret_key,
        algorithm=settings.jwt_algorithm,
    )

def decode_token(token: str) -> dict:
    return jwt.decode(token, settings.jwt_secret_key, algorithms=[settings.jwt_algorithm])

def generate_api_key() -> tuple[str, str]:
    """Returns (raw_key, key_hash). Raw key is shown once; hash is stored."""
    random_part = secrets.token_urlsafe(32)
    raw_key = f"{API_KEY_PREFIX}{random_part}"
    key_hash = hashlib.sha256(raw_key.encode()).hexdigest()
    return raw_key, key_hash

def hash_api_key(raw_key: str) -> str:
    return hashlib.sha256(raw_key.encode()).hexdigest()
```

### RBAC Middleware (core/permissions.py)

```python
from functools import wraps
from fastapi import HTTPException, status
from app.core.constants import ROLE_PERMISSIONS

def require_permission(resource: str, action: str):
    """Dependency that checks if the current user has the required permission."""
    def checker(auth_context = Depends(get_auth_context)):
        permission = f"{resource}:{action}"
        if permission not in auth_context.permissions:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"Permission denied: {permission}",
            )
        return auth_context
    return checker

def require_role(*roles: str):
    """Dependency that checks if the current user has one of the required roles."""
    def checker(auth_context = Depends(get_auth_context)):
        if auth_context.role not in roles:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"Role required: {', '.join(roles)}",
            )
        return auth_context
    return checker

def require_owner_or_admin(resource_owner_id_param: str = "owner_id"):
    """Allow access if user is admin or owns the resource."""
    def checker(auth_context = Depends(get_auth_context), **kwargs):
        if auth_context.role == "ADMIN":
            return auth_context
        # The route handler must verify ownership using auth_context.user_id
        return auth_context
    return checker
```

### Auth Dependency (dependencies.py)

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from app.core.security import decode_token, hash_api_key, API_KEY_PREFIX
from app.schemas.auth import AuthContext

bearer_scheme = HTTPBearer(auto_error=False)

async def get_auth_context(
    credentials: HTTPAuthorizationCredentials | None = Depends(bearer_scheme),
    session = Depends(get_session),
) -> AuthContext:
    if not credentials:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Not authenticated")

    token = credentials.credentials

    # API Key auth
    if token.startswith(API_KEY_PREFIX):
        key_hash = hash_api_key(token)
        api_key = await api_key_repo.find_by_hash(session, key_hash)
        if not api_key or not api_key.is_active:
            raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Invalid API key")
        if api_key.expires_at and api_key.expires_at < datetime.now(timezone.utc):
            raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="API key expired")

        user = await user_repo.find_by_id(session, api_key.user_id)
        permissions = await permission_repo.get_permissions_for_role(session, user.role)
        return AuthContext(
            user_id=str(user.id),
            email=user.email,
            role=user.role,
            permissions=permissions,
            auth_method="api_key",
            api_key_id=str(api_key.id),
        )

    # JWT auth
    try:
        payload = decode_token(token)
        if payload.get("type") != "access":
            raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Invalid token type")
        return AuthContext(
            user_id=payload["sub"],
            email=payload["email"],
            role=payload["role"],
            permissions=payload.get("permissions", []),
            auth_method="jwt",
        )
    except JWTError:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Invalid token")
```

---

## JavaScript Implementation

### Security Utilities (core/security.ts)

```typescript
import { createHash, randomBytes } from "crypto";
import bcrypt from "bcryptjs";
import jwt from "jsonwebtoken"; // or use @fastify/jwt
import { config } from "../config";

const API_KEY_PREFIX = "aib_sk_";
const SALT_ROUNDS = 12;

export async function hashPassword(password: string): Promise<string> {
  return bcrypt.hash(password, SALT_ROUNDS);
}

export async function verifyPassword(plain: string, hashed: string): Promise<boolean> {
  return bcrypt.compare(plain, hashed);
}

export function createAccessToken(payload: Record<string, unknown>): string {
  return jwt.sign(
    { ...payload, type: "access" },
    config.jwtSecret,
    { expiresIn: config.jwtExpiresIn },
  );
}

export function createRefreshToken(userId: string): string {
  return jwt.sign(
    { sub: userId, type: "refresh" },
    config.jwtSecret,
    { expiresIn: config.jwtRefreshExpiresIn },
  );
}

export function decodeToken(token: string): jwt.JwtPayload {
  return jwt.verify(token, config.jwtSecret) as jwt.JwtPayload;
}

export function generateApiKey(): { rawKey: string; keyHash: string } {
  const randomPart = randomBytes(32).toString("base64url");
  const rawKey = `${API_KEY_PREFIX}${randomPart}`;
  const keyHash = createHash("sha256").update(rawKey).digest("hex");
  return { rawKey, keyHash };
}

export function hashApiKey(rawKey: string): string {
  return createHash("sha256").update(rawKey).digest("hex");
}
```

### Auth Plugin (plugins/auth.plugin.ts)

```typescript
import fp from "fastify-plugin";
import { FastifyInstance, FastifyRequest } from "fastify";
import { decodeToken, hashApiKey } from "../core/security";

const API_KEY_PREFIX = "aib_sk_";

export interface AuthContext {
  userId: string;
  email: string;
  role: "ADMIN" | "USER";
  permissions: string[];
  authMethod: "jwt" | "api_key";
  apiKeyId?: string;
}

declare module "fastify" {
  interface FastifyRequest {
    auth: AuthContext;
  }
}

export const authPlugin = fp(async (app: FastifyInstance) => {
  app.decorateRequest("auth", null);

  app.decorate("authenticate", async (request: FastifyRequest) => {
    const authHeader = request.headers.authorization;
    if (!authHeader?.startsWith("Bearer ")) {
      throw app.httpErrors.unauthorized("Missing authorization header");
    }

    const token = authHeader.slice(7);

    // API Key auth
    if (token.startsWith(API_KEY_PREFIX)) {
      const keyHash = hashApiKey(token);
      const apiKey = await request.server.apiKeyRepo.findByHash(keyHash);
      if (!apiKey || !apiKey.isActive) {
        throw app.httpErrors.unauthorized("Invalid API key");
      }
      if (apiKey.expiresAt && apiKey.expiresAt < new Date()) {
        throw app.httpErrors.unauthorized("API key expired");
      }

      const user = await request.server.userRepo.findById(apiKey.userId);
      const permissions = await request.server.permissionRepo.getForRole(user.role);
      request.auth = {
        userId: user.id,
        email: user.email,
        role: user.role,
        permissions,
        authMethod: "api_key",
        apiKeyId: apiKey.id,
      };
      return;
    }

    // JWT auth
    try {
      const payload = decodeToken(token);
      if (payload.type !== "access") {
        throw app.httpErrors.unauthorized("Invalid token type");
      }
      request.auth = {
        userId: payload.sub as string,
        email: payload.email as string,
        role: payload.role as "ADMIN" | "USER",
        permissions: (payload.permissions as string[]) ?? [],
        authMethod: "jwt",
      };
    } catch {
      throw app.httpErrors.unauthorized("Invalid token");
    }
  });
});
```

### RBAC Middleware (middleware/auth.middleware.ts)

```typescript
import { FastifyRequest, FastifyReply } from "fastify";

export function requirePermission(resource: string, action: string) {
  return async (request: FastifyRequest, reply: FastifyReply) => {
    await request.server.authenticate(request);
    const permission = `${resource}:${action}`;
    if (!request.auth.permissions.includes(permission)) {
      throw request.server.httpErrors.forbidden(`Permission denied: ${permission}`);
    }
  };
}

export function requireRole(...roles: string[]) {
  return async (request: FastifyRequest, reply: FastifyReply) => {
    await request.server.authenticate(request);
    if (!roles.includes(request.auth.role)) {
      throw request.server.httpErrors.forbidden(`Role required: ${roles.join(", ")}`);
    }
  };
}

export function requireAuth() {
  return async (request: FastifyRequest, reply: FastifyReply) => {
    await request.server.authenticate(request);
  };
}
```

---

## Database Seed Script

### Python (scripts/seed.py)

```python
import asyncio
from app.db.session import async_session_factory
from app.models.user import User
from app.models.permission import Permission
from app.core.security import hash_password
from app.core.constants import ADMIN_PERMISSIONS, USER_PERMISSIONS

async def seed():
    async with async_session_factory() as session:
        # Create admin user
        admin = User(
            email="admin@example.com",
            password_hash=hash_password("changeme"),
            name="Admin",
            role="ADMIN",
            is_active=True,
        )
        session.add(admin)

        # Seed permissions
        for perm in ADMIN_PERMISSIONS:
            resource, action = perm.split(":")
            session.add(Permission(role="ADMIN", resource=resource, action=action))
        for perm in USER_PERMISSIONS:
            resource, action = perm.split(":")
            session.add(Permission(role="USER", resource=resource, action=action))

        await session.commit()
        print("Seed complete. Admin: admin@example.com / changeme")

if __name__ == "__main__":
    asyncio.run(seed())
```

### JavaScript (src/db/seed.ts)

```typescript
import { hashPassword } from "../core/security";
import { ADMIN_PERMISSIONS, USER_PERMISSIONS } from "../core/constants";

export async function seed(prisma: PrismaClient) {
  // Create admin user
  const admin = await prisma.user.upsert({
    where: { email: "admin@example.com" },
    update: {},
    create: {
      email: "admin@example.com",
      passwordHash: await hashPassword("changeme"),
      name: "Admin",
      role: "ADMIN",
      isActive: true,
    },
  });

  // Seed permissions
  const allPerms = [
    ...ADMIN_PERMISSIONS.map((p) => ({ ...parsePermission(p), role: "ADMIN" as const })),
    ...USER_PERMISSIONS.map((p) => ({ ...parsePermission(p), role: "USER" as const })),
  ];

  for (const perm of allPerms) {
    await prisma.permission.upsert({
      where: { role_resource_action: { role: perm.role, resource: perm.resource, action: perm.action } },
      update: {},
      create: perm,
    });
  }

  console.log("Seed complete. Admin: admin@example.com / changeme");
}

function parsePermission(perm: string) {
  const [resource, action] = perm.split(":");
  return { resource, action };
}
```

### Constants (core/constants)

```python
# Python
from enum import StrEnum

class Role(StrEnum):
    ADMIN = "ADMIN"
    USER = "USER"

ADMIN_PERMISSIONS = [
    "users:create", "users:read", "users:update", "users:delete",
    "api_keys:create", "api_keys:read", "api_keys:revoke",
    "documents:create", "documents:read", "documents:update", "documents:delete",
    "search:query", "llm:generate", "admin:dashboard",
]

USER_PERMISSIONS = [
    "users:read",  # own profile
    "users:update",  # own profile
    "api_keys:create", "api_keys:read", "api_keys:revoke",  # own keys
    "documents:create", "documents:read", "documents:update", "documents:delete",  # own docs
    "search:query", "llm:generate",
]
```

```typescript
// TypeScript
export const Role = { ADMIN: "ADMIN", USER: "USER" } as const;
export type Role = (typeof Role)[keyof typeof Role];

export const ADMIN_PERMISSIONS = [
  "users:create", "users:read", "users:update", "users:delete",
  "api_keys:create", "api_keys:read", "api_keys:revoke",
  "documents:create", "documents:read", "documents:update", "documents:delete",
  "search:query", "llm:generate", "admin:dashboard",
] as const;

export const USER_PERMISSIONS = [
  "users:read", "users:update",
  "api_keys:create", "api_keys:read", "api_keys:revoke",
  "documents:create", "documents:read", "documents:update", "documents:delete",
  "search:query", "llm:generate",
] as const;
```
