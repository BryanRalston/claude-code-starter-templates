# [YOUR_PROJECT_NAME] — Backend API Service

<!-- WHY THIS EXISTS: Claude needs to understand API architecture patterns to generate consistent,
     production-quality endpoints. Replace all [BRACKETED] values with your project details. -->

[YOUR_PROJECT_NAME] is a [REST | GraphQL] API that powers [describe what it serves — e.g., "the mobile and web clients for a food delivery platform"].

## Tech Stack

- **Runtime**: [Node.js 22 LTS | Python 3.12]
- **Framework**: [Express | Fastify | FastAPI | Django REST Framework]
- **Language**: [TypeScript 5.x (strict) | Python with type hints]
- **Database**: [PostgreSQL 16 | MySQL 8 | MongoDB 7] via [Prisma | Drizzle | SQLAlchemy | Django ORM]
- **Cache**: [Redis 7 | Valkey] — sessions, rate limiting, hot queries
- **Queue**: [BullMQ | Celery | SQS] — background jobs, email, webhooks
- **Auth**: [JWT (access + refresh) | API keys | OAuth 2.0]
- **Validation**: [Zod | Joi | Pydantic]
- **Docs**: [OpenAPI/Swagger auto-generated from route definitions]

## API Conventions

<!-- WHY: Without explicit conventions, Claude generates inconsistent endpoint patterns. -->

**Base URL**: `https://api.[YOUR_DOMAIN].com/v1`

**Versioning**: URL path prefix (`/v1/`, `/v2/`). Major version only. Breaking changes = new version.

**Resource naming**: Plural nouns, kebab-case. `GET /v1/team-members`, not `/v1/getTeamMember`.

**Standard response envelope**:
```json
{
  "data": { ... },           // Single object or array
  "meta": {                  // Pagination, counts (list endpoints only)
    "total": 142,
    "page": 1,
    "per_page": 25
  },
  "error": null              // Error object when status >= 400
}
```

**Standard error format**:
```json
{
  "data": null,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Human-readable description",
    "details": [              // Field-level errors for 422
      { "field": "email", "message": "Already registered" }
    ]
  }
}
```

## Route / Endpoint Structure

<!-- WHY: Claude needs to know where to create new endpoints. -->

```
src/
  routes/              # Route definitions (thin — delegate to services)
    v1/
      auth.ts          # POST /login, /register, /refresh, /logout
      users.ts         # CRUD for users
      [resource].ts    # Each resource gets its own file
  services/            # Business logic (testable, framework-agnostic)
  repositories/        # Database access layer (queries, transactions)
  middleware/          # Auth, validation, rate-limit, error handler, logging
  jobs/               # Background job processors
  lib/                # Shared utilities, config, constants
  types/              # TypeScript types / Pydantic models
```

- **Routes** call **Services**, Services call **Repositories**. Never skip layers.
- One service file per domain concept. One repository per database table/collection.

## Auth Middleware

<!-- WHY: Auth is the #1 source of security bugs in generated code. Be explicit. -->

- **Public routes**: Listed explicitly in middleware config. Everything else requires auth.
- **JWT flow**: Access token (15min, in Authorization header) + Refresh token (7d, httpOnly cookie).
- **API keys**: For service-to-service. Passed via `X-API-Key` header. Scoped to specific endpoints.
- **RBAC roles**: `[admin | member | viewer]` — checked via `requireRole('admin')` middleware.
- **Rate limiting**: [100 req/min per IP for public, 1000 req/min per user for authenticated].

## Database

<!-- WHY: Claude needs migration workflow and key relationships to write correct data access. -->

**Key tables**: `users`, `organizations`, `[your_core_table]`, `[your_secondary_table]`

- `organizations` -> `users` (many-to-many via `memberships` with `role` column)
- [YOUR_CORE_TABLE] belongs to `organizations`, has [describe relationships]
- All tables have: `id` (UUID), `created_at`, `updated_at`
- Soft deletes: `deleted_at` on [list tables]

**Migration commands**:
```bash
npm run db:migrate:create   # Generate new migration file
npm run db:migrate:run      # Apply pending migrations
npm run db:migrate:rollback # Revert last migration
npm run db:seed             # Seed dev data
```

## Quick Commands

```bash
npm run dev              # Start with hot reload (port 3000)
npm run build            # Compile TypeScript
npm run start            # Run production build
npm run test             # Unit + integration tests
npm run test:e2e         # End-to-end API tests (requires running server)
npm run lint             # ESLint / Ruff
npm run typecheck        # TypeScript strict check / mypy
npm run db:studio        # Database GUI
docker compose up -d     # Start DB + Redis + dependencies
```

## Testing

<!-- WHY: Claude generates much better tests when it knows the layered testing strategy. -->

- **Unit tests**: Services + utilities. Mock repositories. Fast, no DB.
- **Integration tests**: Routes through full stack with test database. Reset DB between suites.
- **E2E tests**: Full API flows (register -> login -> create resource -> query). Run against docker compose.
- **Test files**: Colocated as `[name].test.ts` or in `__tests__/` directory.
- **Factories**: `test/factories/` — use to create test data consistently.

## Docker

```bash
docker compose up -d            # Start all services (DB, Redis, API)
docker compose up -d db redis   # Start dependencies only (for local dev)
docker build -t [your-app] .    # Production image
```

## Environment Variables

<!-- WHY: Claude must know what env vars exist without seeing actual values. -->

```bash
# .env (NEVER committed)
NODE_ENV="development"
PORT=3000
DATABASE_URL="postgresql://user:pass@localhost:5432/[db_name]"
REDIS_URL="redis://localhost:6379"
JWT_SECRET="..."                       # Token signing
JWT_REFRESH_SECRET="..."               # Refresh token signing
[PROVIDER]_API_KEY="..."               # Third-party service keys
SMTP_HOST="..."                        # Transactional email
SENTRY_DSN="..."                       # Error tracking
```

## Performance Targets

| Metric | Target | Notes |
|--------|--------|-------|
| p50 response | < 50ms | Standard CRUD endpoints |
| p99 response | < 500ms | Including complex queries |
| Throughput | > 1000 req/s | Per instance, load-tested |
| DB query time | < 20ms p95 | Add indexes if exceeded |
| Error rate | < 0.1% | 5xx errors in production |

## Common Gotchas

<!-- WHY: These are real production issues. Claude seeing them prevents costly mistakes. -->

1. **N+1 queries**: Always eager-load relations. Use `include`/`joinedLoad`/`select_related`. Review every list endpoint.
2. **Connection pooling**: Never create DB connections per request. Use the ORM's built-in pool (Prisma: `connection_limit` in URL, SQLAlchemy: `pool_size`).
3. **CORS in production**: Allowlist specific origins. Never use `origin: '*'` with credentials. Preflight requests (`OPTIONS`) must also return correct headers.
4. **Transaction boundaries**: Wrap multi-table writes in a transaction. Partial writes = data corruption.
5. **Pagination defaults**: Always set a `per_page` max (100). Unbounded queries will take down your DB.
6. **Request validation at the edge**: Validate input in middleware/route BEFORE it reaches service layer. Never trust client data.
7. **Sensitive data in logs**: Never log passwords, tokens, or full request bodies. Redact sensitive fields.
8. **Graceful shutdown**: Handle `SIGTERM` — drain connections, finish in-flight requests, close DB pool. Critical for container orchestration.
9. **Idempotency**: POST endpoints that create resources should support `Idempotency-Key` header to prevent duplicate creation on retry.
10. **Timezone handling**: Store everything as UTC in the database. Convert to user timezone only at the API response layer.
