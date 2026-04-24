# Backend Module Pattern Guide

## Base Module Structure

Each feature module lives under `src/modules/{moduleName}/` and usually starts with these five files:

1. **{module}.routes.ts** - route registration
2. **{module}.handler.ts** - HTTP request handlers
3. **{module}.service.ts** - business logic and orchestration
4. **{module}.repo.ts** - database queries
5. **{module}.schema.ts** - Zod validation schemas and inferred types

This is the default shape, not a hard rule. Several modules in this codebase add focused support files when the domain is more complex than a basic CRUD flow.

## File Patterns

### Routes File Pattern
```typescript
import type { Router } from "../../shared";
import { {module}Handler } from "./{module}.handler";
import { requireAuth } from "../../middlewares/auth";
import { requireAdminAuth } from "../../middlewares/admin";

export function register{Module}Routes(router: Router) {
  router.get("/path", requireAuth((req, userId) => handler.method(req, userId)));
  router.post("/path", requireAdminAuth((req, adminUserId, params) => handler.method(req, params)));
}
```

### Handler File Pattern
```typescript
import { json, error, handleError } from "../../shared";
import { {module}Service } from "./{module}.service";
import { {schema} } from "./{module}.schema";
import type { RouteParams } from "../../middlewares/auth";

export const {module}Handler = {
  async methodName(req: Request, userId?: number, params?: RouteParams) {
    try {
      const body = await req.json();
      const validatedData = schema.parse(body);
      const result = await {module}Service.methodName(validatedData);
      return json(result, 201);
    } catch (err) {
      const { message, statusCode } = handleError(err);
      return error(message, statusCode);
    }
  }
};
```

### Service File Pattern
```typescript
import { {module}Repository } from "./{module}.repo";
import { NotFoundError, ValidationError } from "../../shared";

export class {Module}Service {
  async methodName(data: InputType) {
    const result = await {module}Repository.method(data);
    if (!result) throw new NotFoundError("Not found");
    return result;
  }
}

export const {module}Service = new {Module}Service();
```

### Repository File Pattern
```typescript
import { prisma } from "../../providers/database";

export class {Module}Repository {
  async findAll(page?: number, limit?: number) {
    return prisma.{model}.findMany({
      include: { /* relations */ },
      skip: (page - 1) * limit,
      take: limit
    });
  }

  async findById(id: number) {
    return prisma.{model}.findUnique({ where: { id } });
  }
}

export const {module}Repository = new {Module}Repository();
```

Some modules intentionally replace the repository file with a more specific collaborator:

- `ai.logger.ts` for usage-log persistence that does not need a full repository surface
- `upload.storage.ts` for S3-specific storage operations
- `revenuecat.client.ts` for third-party billing API calls

Use a specialized file when the concern is not general-purpose data access.

### Schema File Pattern
```typescript
import { z } from "zod";

export const createSchema = z.object({
  field: z.string().min(1, "Required")
});

export const updateSchema = z.object({
  field: z.string().optional()
});

export type CreateInput = z.infer<typeof createSchema>;
export type UpdateInput = z.infer<typeof updateSchema>;
export interface Response { /* ... */ }
```

## Pattern Variations

Use extra module files only when they give a cleaner boundary than enlarging the service or handler.

### `.constants.ts`

Use for prompts, limits, reusable messages, lookup tables, and module-scoped configuration.

Current examples:

- `auth.constants.ts`
- `ai-coach.constants.ts`
- `upload.constants.ts`

### `.types.ts`

Use when schemas alone do not cover a module's shared types or when multiple files reuse larger domain interfaces.

Current example:

- `ai-coach.types.ts`

### `.validator.ts` or `.validation.ts`

Use when a module needs post-parse validation for AI outputs, file uploads, or domain-specific constraints.

Current examples:

- `ai-coach.validator.ts`
- `workout.validator.ts`
- `upload.validation.ts`

### `.presenter.ts`

Use when response shaping should stay separate from service logic.

Current example:

- `admin.presenter.ts`

### `.gateway.ts` or `.client.ts`

Use for third-party APIs or external-service coordination.

Current examples:

- `auth.gateway.ts`
- `subscriptions/revenuecat.client.ts`

### `.storage.ts`

Use when storage behavior has enough logic to deserve its own boundary.

Current example:

- `upload.storage.ts`

### `.logger.ts`

Use when a module needs write-only analytics or audit persistence without creating a full repository abstraction.

Current example:

- `ai.logger.ts`

## Current Module Variants

Modules that extend the base pattern today:

- `auth`: admin auth routes, token helpers, auth gateway, constants
- `ai-coach`: constants, types, validator, richer repository interactions
- `uploads`: storage, constants, and upload validation helpers
- `subscriptions`: RevenueCat client plus multiple service surfaces in one module
- `admins`: presenter layer for admin-facing response formatting

Prefer these extensions only when the underlying responsibility is genuinely separate.

## Core Patterns

### Module Registration (app.ts)
All modules registered in one place:
```typescript
import { register{Module}Routes } from "./modules/{module}/{module}.routes";
register{Module}Routes(router);
```

### Authentication Patterns
- **`requireAuth(handler)`** - Standard user auth (userId required)
- **`requireAdminAuth(handler)`** - Admin-only (checks admin role in DB + JWT)

### Error Handling
- Use custom error classes: `ValidationError`, `NotFoundError`, `UnauthorizedError`, `ConflictError`
- Handler template: `const { message, statusCode } = handleError(err);`
- Uses `ErrorTypes` enum for predefined messages

### Response Patterns
- Success: `json(data, statusCode)` - default 200, can specify 201
- Error: `error(message, statusCode)`
- Both accept optional `headers` in options

### Validation Pattern
- Import `{schema}` from `./{module}.schema`
- Validate: `const data = schema.parse(body)`
- Zod handles both validation + type inference

### Caching Pattern
```typescript
const cached = await cache.get(key);
if (cached) return cached;
const result = await repo.method();
await cache.set(key, result, CACHE.TTL_SECONDS);
```

Caching is currently used for low-churn lookup or profile data such as muscles, equipment, exercises, and some subscription checks.

The cache abstraction is async and driver-based:

- `src/shared/cache/` exposes the application cache surface and cache-key helpers
- `src/providers/redis/redis.client.ts` provides the Redis client when Redis is configured
- the app uses Redis when `CACHE_DRIVER=redis` or when auto mode has a Redis URL available; otherwise it falls back to in-memory cache
- prefer `invalidateKey()` and `invalidateNamespace()` after writes instead of mutating cached objects in place

### Database Transactions
```typescript
await prisma.$transaction(async (tx) => {
  await tx.model.create({ data });
  await tx.model.update({ data });
});
```

## Key Infrastructure Files

- **src/shared/http/router.ts** - Custom Router (no Express) with pattern matching
- **src/providers/database/prisma.client.ts** - Prisma client singleton
- **src/shared/http/response.ts** - json() & error() helpers with BigInt support
- **src/shared/validation/common.ts** - Validation utility functions
- **src/shared/errors/app-error.ts** - Custom error classes + handleError()
- **src/shared/logging/logger.ts** - Centralized logging
- **src/shared/cache/** - Cache abstraction and cache-key helpers
- **src/shared/pagination/pagination.ts** - Pagination parsing and paginated responses
- **src/shared/presentation/** - Shared response presenters
- **src/providers/aws/s3.client.ts** - AWS S3 provider
- **src/providers/aws/ses.client.ts** - AWS SES provider
- **src/providers/google/google-auth.client.ts** - Google auth provider
- **src/providers/apple/apple-auth.client.ts** - Apple auth provider
- **src/providers/redis/redis.client.ts** - Redis cache provider
- **src/middlewares/auth.ts** - requireAuth() wrapper
- **src/middlewares/admin.ts** - requireAdminAuth() wrapper (JWT + DB check)
- **src/middlewares/cors.ts** - CORS preflight + origin headers
- **src/middlewares/rate-limit.ts** - Auth, upload, and general rate limiting
- **src/middlewares/request-limit.ts** - Request size enforcement
- **src/config/env.ts** - Environment variables with validation
- **src/config/constants.ts** - HTTP status codes, size limits, timeouts, cache TTLs
- **src/config/rate-limits.ts** - Per-surface rate-limit config

## Prisma Patterns

- Use `select` for field projection, not in routes/handlers
- Use `include` in repository for relations
- `@relation("name")` for back-references
- `onDelete: Cascade` for cleanup
- `@unique`, `@index` for DB optimization

## Practical Rules

- Keep routes declarative and thin.
- Keep handlers HTTP-specific; avoid business logic there.
- Keep services as the orchestration boundary.
- Keep repositories Prisma-specific.
- Add presenters when formatting logic starts leaking into services.
- Add gateway, client, storage, validator, or logger files only when the concern is distinct enough to justify a separate abstraction.
