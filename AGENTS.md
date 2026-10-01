# AGENTS.md — Wishlist Service

## 1. Stack

| Technology | Role |
|---|---|
| **Node.js 20 LTS + Fastify 4** | Primary runtime and HTTP framework (preferred); Java 21 + Spring Boot 3 is the alternate |
| **TypeScript 5** | Type safety across the entire Node.js codebase |
| **PostgreSQL 15 (AWS RDS)** | Primary wishlist store with row-level optimistic locking; DynamoDB is the alternate for key-value-only deployments |
| **Prisma ORM** | Schema management, migrations, and type-safe DB access (Node.js path) |
| **Kafka.js / AWS SQS SDK v3** | Consuming `UserRegistered` events from the message broker |
| **jsonwebtoken + @fastify/jwt** | JWT validation middleware on every route |
| **Zod** | Request/response schema validation |
| **Axios / Got** | HTTP client for Product Catalog Service validation calls |
| **Jest 29 + Supertest** | Unit and integration testing (Node.js); JUnit 5 + Testcontainers (Java alternate) |
| **Docker + AWS ECS / Kubernetes** | Containerised deployment |
| **GitHub Actions** | CI pipeline |
| **ESLint + Prettier** | Code style enforcement |
| **Pino** | Structured JSON logging (built into Fastify) |

---

## 2. Project Structure

```
wishlist-service/
├── .github/
│   └── workflows/
│       └── ci.yml                  # GitHub Actions CI pipeline
├── prisma/
│   ├── schema.prisma               # Wishlist + WishlistItem models, optimistic-lock version field
│   └── migrations/                 # Auto-generated Prisma migration files
├── src/
│   ├── config/
│   │   └── index.ts                # Centralised env-var loading and validation (Zod)
│   ├── db/
│   │   └── client.ts               # Singleton Prisma client instance
│   ├── plugins/
│   │   ├── jwt.ts                  # @fastify/jwt registration and JWT verify decorator
│   │   ├── sensible.ts             # @fastify/sensible for standard HTTP errors
│   │   └── requestContext.ts       # Per-request context (userId injection)
│   ├── modules/
│   │   └── wishlist/
│   │       ├── wishlist.routes.ts  # Fastify route definitions (add, remove, list)
│   │       ├── wishlist.controller.ts  # Route handlers; delegates to service
│   │       ├── wishlist.service.ts     # Business logic, optimistic locking, product validation
│   │       ├── wishlist.repository.ts  # All DB queries via Prisma
│   │       ├── wishlist.schema.ts      # Zod schemas for request/response validation
│   │       └── wishlist.types.ts       # Shared TypeScript interfaces and DTOs
│   ├── consumers/
│   │   ├── userRegistered.consumer.ts  # Kafka/SQS consumer; calls wishlist.service.initializeWishlist()
│   │   └── consumer.factory.ts         # Selects Kafka or SQS consumer based on config
│   ├── clients/
│   │   └── productCatalog.client.ts    # HTTP client wrapping Product Catalog Service calls
│   ├── middleware/
│   │   └── auth.ts                 # Fastify preHandler: JWT decode + userId ownership check
│   ├── errors/
│   │   └── AppError.ts             # Typed application error classes (NotFound, Forbidden, Conflict)
│   ├── utils/
│   │   └── logger.ts               # Pino logger wrapper
│   ├── app.ts                      # Fastify app factory; registers plugins, routes, hooks
│   └── server.ts                   # Entry point; binds port, starts consumer
├── test/
│   ├── unit/
│   │   ├── wishlist.service.test.ts
│   │   ├── wishlist.repository.test.ts
│   │   └── productCatalog.client.test.ts
│   ├── integration/
│   │   ├── wishlist.routes.test.ts  # Supertest against real Fastify app + test DB
│   │   └── userRegistered.consumer.test.ts
│   ├── fixtures/
│   │   └── wishlistFixtures.ts     # Reusable test data factories
│   └── helpers/
│       └── testApp.ts              # Builds Fastify app wired to test DB
├── docker/
│   └── init.sql                    # Optional DB seed for local dev
├── .env.example                    # All required env vars documented
├── .eslintrc.json
├── .prettierrc
├── docker-compose.yml              # Local dev: postgres, kafka/localstack, service
├── Dockerfile
├── jest.config.ts
├── package.json
├── tsconfig.json
└── tasks.md                        # Agent-generated task checklist (see §3)
```

### Key File Purposes (expanded)

| File | Purpose |
|---|---|
| `prisma/schema.prisma` | Defines `Wishlist` (userId, version) and `WishlistItem` (wishlistId, productId, addedAt) models; `version` field enables optimistic locking |
| `src/config/index.ts` | Parses and validates all env vars at startup; throws on missing required values |
| `src/modules/wishlist/wishlist.service.ts` | Owns all business rules: product ID validation, concurrent-write safety, event-driven init |
| `src/consumers/userRegistered.consumer.ts` | Idempotently creates a `Wishlist` row when a `UserRegistered` event arrives |
| `src/clients/productCatalog.client.ts` | Validates product existence before wishlist add; circuit-breaker pattern recommended |

---

## 3. Required Workflow

The agent **must** follow these steps in order. Do not skip or reorder.

```
STEP 1 — Read all specification documents
  - Read every story/spec file in the repository before writing any code.
  - Identify all acceptance criteria, edge cases, and integration contracts.

STEP 2 — Create tasks.md
  - At the project root, create tasks.md.
  - List every implementation task as an unchecked checkbox: - [ ] <task>
  - Group tasks under headings: Setup, Database, API Routes, Consumer, Client Integrations,
    Auth & Security, Testing, Docker & CI.
  - Do not begin implementation until tasks.md is written and complete.

STEP 3 — Environment & Tooling Setup
  - Initialise npm project, install all dependencies listed in §1.
  - Configure tsconfig.json (strict mode, ES2022 target, moduleResolution: bundler).
  - Configure ESLint (typescript-eslint), Prettier, and jest.config.ts.
  - Copy .env.example; document every variable with a comment.

STEP 4 — Database Schema
  - Write prisma/schema.prisma with Wishlist and WishlistItem models.
  - Include version Int @default(0) on Wishlist for optimistic locking.
  - Run `prisma migrate dev --name init` to generate migration files.
  - Commit migration files — never edit generated migrations manually.

STEP 5 — Core Application Shell
  - Implement src/app.ts (Fastify factory, plugin registration, route registration).
  - Implement src/server.ts (startup, graceful shutdown, consumer start).
  - Implement src/config/index.ts with Zod env validation.

STEP 6 — Auth Middleware
  - Register @fastify/jwt plugin in src/plugins/jwt.ts.
  - Implement src/middleware/auth.ts preHandler:
      * Verify JWT signature and expiry.
      * Extract userId from token claims.
      * On wishlist-scoped routes, assert token userId === route :userId param.
      * Return 401 for invalid/missing token; 403 for ownership mismatch.

STEP 7 — Wishlist Module
  - Implement repository → service → controller → routes in that order.
  - Repository: raw Prisma queries only; no business logic.
  - Service (wishlist.service.ts):
      * addItem: validate productId via productCatalog.client, then upsert with
        conditional version check (optimistic lock); retry up to 3 times on conflict.
      * removeItem: delete WishlistItem; verify ownership before delete.
      * listItems: paginated query (cursor or offset); attach product metadata from cache or client.
      * initializeWishlist: idempotent upsert of Wishlist row for a userId.
  - Controller: parse validated request, call service, return shaped response.
  - Routes: apply auth preHandler to all routes; use Zod schemas for input validation.

STEP 8 — Event Consumer
  - Implement consumer.factory.ts to select KafkaJS or SQS consumer from config.
  - Implement userRegistered.consumer.ts:
      * Parse and validate incoming event payload with Zod.
      * Call wishlistService.initializeWishlist(userId).
      * Commit offset / delete SQS message only after successful processing.
      * Log and dead-letter (DLQ) events that fail after 3 retries.

STEP 9 — Product Catalog Client
  - Implement productCatalog.client.ts using Axios.
  - Validate product existence: GET /products/:productId → 200 means valid.
  - Treat 404 as invalid product (throw AppError.NotFound).
  - Treat 5xx as transient; implement exponential backoff with 3 retries.
  - Add circuit-breaker using opossum or equivalent.

STEP 10 — Tests
  - Write all unit tests in test/unit/ (mock DB and HTTP clients).
  - Write all integration tests in test/integration/ (real Fastify app, real test DB via docker-compose).
  - Achieve ≥ 90% line and branch coverage.
  - Run `npm test` and confirm all tests pass before proceeding.

STEP 11 — Docker & CI
  - Write Dockerfile (multi-stage).
  - Write docker-compose.yml for local dev.
  - Write .github/workflows/ci.yml.
  - Verify `docker build` succeeds locally.

STEP 12 — Final Validation
  - Run `npm run lint` — zero errors.
  - Run `npm test -- --coverage` — ≥ 90% coverage on all metrics.
  - Run `docker build -t wishlist-service .` — exits 0.
  - Check every tasks.md checkbox.
  - Confirm no hardcoded secrets, credentials, or localhost URLs exist in source files.
```

---

## 4. Coding Conventions

### Naming

| Construct | Convention | Example |
|---|---|---|
| Files | `kebab-case` | `wishlist.service.ts` |
| Classes | `PascalCase` | `WishlistService` |
| Interfaces / Types | `PascalCase` prefixed with `I` for interfaces | `IWishlistItem`, `AddItemDto` |
| Functions / methods | `camelCase` | `addItem()`, `validateProduct()` |
| Constants / env keys | `SCREAMING_SNAKE_CASE` | `JWT_SECRET`, `DB_URL` |
| Fastify route params | `camelCase` in schema, `:userId` in path | `/wishlists/:userId/items` |
| Database columns | `snake_case` (Prisma maps to camelCase) | `product_id`, `added_at` |

### Architecture Patterns

- **Strict layering**: Routes → Controller → Service → Repository. No layer may skip another.
- **Dependency injection via constructor**: Pass `db`, `logger`, and `config` into service/repository constructors; never import singletons directly inside classes.
- **No business logic in routes or controllers**: Controllers only parse input, call service, and shape output.
- **Repository returns domain objects**: Never expose raw Prisma types outside the repository layer; map to DTOs.
- **Optimistic locking pattern**:
  ```typescript
  // In wishlist.repository.ts
  async addItem(wishlistId: string, productId: string, expectedVersion: number) {
    return this.db.wishlist.update({
      where: { id: wishlistId, version: expectedVersion },
      data: { version: { increment: 1 }, items: { create: { productId } } },
    });
    // Prisma throws P2025 (record not found) if version mismatch → caller retries
  }
  ```
- **Idempotency**: `initializeWishlist` must use `upsert`; `addItem` must be idempotent on duplicate (productId, wishlistId) pairs.
- **Error handling**: All thrown errors must be instances of `AppError` subclasses. Fastify `setErrorHandler` maps these to HTTP responses.
- **No `any` types**: ESLint rule `@typescript-eslint/no-explicit-any` set to `error`.
- **Async/await only**: No raw Promise chains or callbacks.
- **Pagination**: Use cursor-based pagination by default (`cursor`, `take` params); expose `nextCursor` in response.

### Environment Variables (document all in `.env.example`)

```
DATABASE_URL=postgresql://user:pass@localhost:5432/wishlist
JWT_SECRET=
JWT_ISSUER=
PRODUCT_CATALOG_BASE_URL=http://product-catalog-service
KAFKA_BROKERS=localhost:9092          # or blank to use SQS
SQS_QUEUE_URL=                        # or blank to use Kafka
USER_REGISTERED_TOPIC=user.registered
PORT=3000
LOG_LEVEL=info
NODE_ENV=development
```

---

## 5. Testing

### Structure

```
test/
├── unit/                            # Pure logic; all external deps mocked
│   ├── wishlist.service.test.ts     # All service methods, all branches
│   ├── wishlist.repository.test.ts  # Repository query construction
│   └── productCatalog.client.test.ts  # Retry, circuit-breaker, error mapping
├── integration/
│   ├── wishlist.routes.test.ts      # Full HTTP round-trip via Supertest
│   └── userRegistered.consumer.test.ts  # Consumer processes event → DB row created
├── fixtures/
│   └── wishlistFixtures.ts          # Factory functions for test data
└── helpers/
    └── testApp.ts                   # Creates Fastify app bound to test DB
```

### Unit Test Rules

- Mock Prisma using `jest.mock` or `jest-mock-extended`.
- Mock `productCatalog.client.ts` with `jest.fn()`.
- Every public service method must have tests for: happy path, validation failure, product-not-found, DB conflict (optimistic lock retry), ownership violation.
- Use `describe` / `it` blocks; each `it` tests exactly one behaviour.

### Integration Test Rules

- Use a real PostgreSQL instance started via `docker-compose` (or Testcontainers).
- Run `prisma migrate deploy` before the test suite in `globalSetup`.
- Truncate tables in `beforeEach` to ensure test isolation.
- Use `supertest(app.server)` — never start a real HTTP server on a port.
- Test JWT auth: include a helper `makeJwt(userId)` that signs a token with the test secret.
- Assert HTTP status codes, response body shape, and DB state after mutations.

### Coverage Requirements

```
# jest.config.ts
coverageThreshold: {
  global: {
    lines: 90,
    branches: 90,
    functions: 90,
    statements: 90,
  },
},
```

### Running Tests

```bash
# Unit tests only
npm run test:unit

# Integration tests (requires docker-compose up -d)
npm run test:integration

# All tests with coverage report
npm test -- --coverage

# Watch mode during development
npm run test:watch
```

### package.json Scripts

```json
{
  "scripts": {
    "test": "jest",
    "test:unit": "jest --testPathPattern=test/unit",
    "test:integration": "jest --testPathPattern=test/integration --runInBand",
    "test:watch": "jest --watch",
    "