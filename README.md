# AI SaaS Backend

A production-grade REST API backend for an AI-powered SaaS platform. Built to demonstrate real-world backend engineering skills — from secure JWT authentication and Redis-backed session management to OpenAI integration with cost tracking, vector embeddings, Retrieval-Augmented Generation (RAG), multi-tenant organisations, plan-based billing, and webhook delivery.

This project was built independently as part of a structured learning journey toward becoming an AI-integrated backend architect.

---

## What Problem It Solves

Most AI SaaS systems stop at "call the OpenAI API and return the result." This project goes further by solving the production concerns that actually matter:

- **Who can call the AI?** — Full JWT auth with refresh token rotation and Redis-based token invalidation
- **How much does it cost per user?** — Per-request token and USD cost tracking stored in PostgreSQL
- **How do you search large documents intelligently?** — pgvector-powered semantic search with overlapping text chunks and cosine similarity scoring
- **How do you answer questions grounded in user documents?** — A full RAG pipeline: embed → store → retrieve → prompt → respond
- **How do you prevent redundant AI calls?** — Semantic caching: embed each question and skip OpenAI entirely when a semantically equivalent question was answered recently
- **How do you prevent abuse?** — Redis-backed rate limiting at both global and auth-specific levels, plus plan-tier request/token/document limits enforced per user
- **How do you offload slow work?** — BullMQ background queues for email and document processing jobs
- **How do callers know when background jobs finish?** — DB-persisted job events table with a polling endpoint and optional HTTP webhook delivery on completion
- **How do you serve multiple teams from one deployment?** — Multi-tenant organisation model: users belong to an org, all data is org-scoped, and org admins can invite members
- **How do you monetise tiered access?** — Free / Pro / Enterprise plan system with per-plan request, token, and document limits; simulated billing events with a Stripe integration pathway
- **Is the system healthy right now?** — Deep health checks for PostgreSQL, Redis, and memory; a metrics endpoint; and a performance endpoint tracking slow requests and top endpoints

---

## Tech Stack

| Technology | Role | Why |
|---|---|---|
| **Node.js + Express 5** | HTTP server | Lightweight, non-blocking I/O; Express 5 adds native async error propagation |
| **PostgreSQL + pgvector** | Primary database + vector store | Single database for relational data and high-dimensional embedding search |
| **Prisma ORM** | Database access | Type-safe queries, schema-as-code, clean migration workflow |
| **Redis** | Cache + rate limit store | Sub-millisecond access token caching; atomic TTL operations for session invalidation; semantic vector index storage |
| **BullMQ** | Background job queues | Redis-backed, reliable job processing with retry support |
| **OpenAI SDK v6** | AI completions + embeddings | GPT-4o-mini for text operations; `text-embedding-3-small` for 1536-dim semantic vectors |
| **JWT (HS256)** | Auth tokens | Stateless access tokens (15m) + stateful refresh tokens (7d) stored in DB |
| **bcryptjs** | Password hashing | Salted hashing with configurable rounds |
| **Joi** | Request validation | Declarative schema validation at the route layer before any business logic runs |
| **Helmet + HPP** | Security headers | HTTP security headers and HTTP parameter pollution protection |
| **Swagger UI** | API documentation | Auto-generated interactive docs served at `/api/docs` |

---

## Architecture

### Request Pipeline

```
Request
  → Rate Limiter (Redis-backed, global 100/15m + auth 10/15m)
  → Logger Middleware
  → Auth Middleware (JWT + token_nbf check)
  → Validation Middleware (Joi)
  → Plan Limit Middleware (daily request count via Redis)
  → Route → Controller → Service
  → PostgreSQL (Prisma) / Redis / OpenAI
  → sendSuccess() → Standardised JSON Response
```

### Token Refresh and Invalidation Strategy

The auth system uses a dual-token model with Redis-based invalidation:

- **Access token** — short-lived (15 min), cached in Redis under `access_token:{id}`
- **Refresh token** — long-lived (7 days), stored in PostgreSQL
- **`token_nbf:{id}`** — a Redis key holding a "not-before" timestamp in milliseconds. On logout or token refresh, this is updated to the new token's `iat * 1000`. The `authenticate` middleware rejects any access token whose `iat * 1000 < nbf`, instantly invalidating previously issued tokens without a database round trip.
- **Idempotent login** — if a valid session already exists, the server returns the existing tokens with `loginStatus: 2` rather than creating duplicates.

### Multi-Tenant Organisation Model

Every data table (`documents`, `embeddings`, `ai_usage`, `rag_history`, `billing_events`, `job_events`) carries an optional `org_id`. The `attachOrg` middleware resolves the requesting user's `orgId` from the DB and attaches it to `req.orgId`. Services write `orgId` when creating records, allowing future queries to be scoped to an organisation. The org creator is automatically promoted to `admin` role in a single Prisma transaction.

### Background Job Design

BullMQ queues (`emailQueue`, `documentQueue`) are defined in `src/config/queue.js`. Workers in `src/workers/` consume these queues asynchronously. On each job completion or failure:

1. The worker calls `updateJobEvent()` to persist the outcome to the `job_events` PostgreSQL table.
2. If the job payload contained a `callbackUrl`, `deliverWebhook()` sends an HTTP POST to that URL with a `job.completed` payload (10-second timeout). The `callbackSent` and `callbackSentAt` fields are updated on the `JobEvent` record regardless of delivery success.

This separates BullMQ's in-memory state (ephemeral) from durable job history (PostgreSQL), enabling both polling and push delivery.

### Semantic Cache for RAG

Before each RAG query hits OpenAI:

1. The question is embedded into a 1536-dim vector.
2. That vector is compared (cosine similarity) against a per-user vector index stored in Redis (`rag:segments:{userId}:vectors`), holding up to 100 recent question vectors.
3. If any cached question scores ≥ 0.85 similarity, the previous answer is returned immediately — no OpenAI call.
4. On a cache miss, the answer is computed, stored, and the new question vector is added to the index (oldest entry evicted when the limit is exceeded).

### Prompt Engineering Layer

All prompts are centralised in `src/utils/promptBuilder.js`. Each builder returns a `{ system, user }` pair:

- **Analyzer** — enforces a strict JSON schema: `summary`, `keyPoints`, `sentiment`, `topics`, `wordCount`
- **Summarizer** — returns `summary`, `bulletPoints`, `readingTimeMinutes`
- **Classifier** — returns `category`, `confidence`, `reasoning`, constrained to caller-provided categories
- **RAG** — injects retrieved document chunks with source labels and relevance scores; instructs the model to answer only from provided sources and never hallucinate

A three-pass `parseAIJson()` utility handles cases where the model wraps output in markdown code fences despite being instructed not to: direct parse → markdown fence stripping → regex extraction of the first `{...}` block.

### Vector Search and RAG Pipeline

```
User uploads document
    → chunkText() splits into overlapping 400-word chunks (80-word overlap)
      with paragraph-aware splitting to preserve semantic units
    → generateEmbeddings() batches all chunks in one OpenAI API call
    → Chunks stored in PostgreSQL via pgvector (raw SQL for vector insertion)

User asks a question
    → Semantic cache check: embed question, cosine-compare against Redis vector index
    → Cache hit: return previous answer (no OpenAI call)
    → Cache miss:
        → pgvector cosine similarity search (<=> operator) retrieves top-k chunks
        → Confidence scored: 70% top-chunk similarity + 30% average similarity
        → buildRAGPrompt() injects chunks as numbered sources with relevance scores
        → OpenAI generates a grounded answer (temperature 0.1 for factual accuracy)
        → Q&A pair saved to rag_history for auditability
        → Token usage tracked and USD cost calculated per request
        → Question vector stored in Redis semantic cache index
```

---

## Key Features and Modules

### Authentication (`/api/v1/auth`)
- Register, login, logout, and token refresh
- Refresh token rotation with database persistence and expiry enforcement
- Redis-based access token invalidation on logout (token_nbf pattern)
- Idempotent login — detects active sessions and returns existing tokens
- `GET /users/me` session guard: returns `401` if the user has logged out, even if the JWT has not yet expired

### Multi-Tenant Organisations (`/api/v1/organisations`)
- Create an organisation — creator is promoted to `admin` role in a single transaction
- Auto-generated URL-safe slugs with deduplication (e.g. `acme-corp-2`)
- Admin-only member list and invite-by-email endpoints
- All data tables carry `org_id` so org-scoped queries are possible without joins
- `attachOrg` middleware resolves `orgId` from the DB and attaches it to `req.orgId`

### Plan-Based Limits + Billing (`/api/v1/billing`)
- Three tiers — **Free**, **Pro**, **Enterprise** — with per-plan daily request limits, monthly token caps, and document quotas
- `enforceRequestLimit` middleware gates every AI endpoint: checks and increments a per-user Redis counter with a midnight expiry
- Monthly token usage checked against plan cap before each OpenAI call
- `POST /billing/upgrade` records a `BillingEvent` and sets `planExpiresAt` (30 days); simulated for now with a clear Stripe integration pathway
- `GET /billing/plans` returns current plan details, available tiers, and pricing table
- `GET /billing/history` returns a chronological log of all plan change events

### AI Text Operations (`/api/v1/ai`)
- `POST /analyze` — sentiment analysis, key points, topic extraction, word count
- `POST /summarize` — concise summary with bullet points and estimated reading time
- `POST /classify` — multi-class classification with caller-defined categories and confidence scores
- Automatic retry with exponential backoff via `withRetry()` wrapper
- Structured JSON parsing with three-pass fallback strategy

### Document Management (`/api/v1/documents`)
- Create, list, retrieve, and delete documents
- AI analysis cached in the document record — repeat requests return instantly without a second OpenAI call
- Document status lifecycle: `pending` → `processing` → `completed` / `failed`
- On failure, status is rolled back so users can retry
- Document count enforced against plan tier before creation

### Semantic Embeddings (`/api/v1/documents/:id/embed`)
- Chunks documents into overlapping segments with paragraph-aware splitting
- Batches all chunks in a single OpenAI `text-embedding-3-small` API call
- Stores 1536-dimensional vectors in PostgreSQL via pgvector
- Tracks character positions (`char_start`, `char_end`) per chunk for future source highlighting

### Vector Search (`/api/v1/search`)
- Semantic similarity search across all user documents or scoped to a single document
- Configurable similarity threshold and result count
- Returns ranked chunks with similarity scores, document title, and text excerpts

### RAG Question Answering (`/api/v1/rag`)
- Natural language questions answered from the user's own document corpus
- Semantic cache checked first — identical-intent questions answered without calling OpenAI
- Confidence levels: `high` / `medium` / `low` / `none` based on similarity score distribution
- Response includes answer, confidence, source citations with relevance scores, and token usage
- Q&A history persisted for audit trail

### Cost-Aware Usage Tracking (`/api/v1/usage`)
- Every OpenAI call records prompt tokens, completion tokens, total tokens, and calculated USD cost
- Per-model pricing table with base-model normalisation — handles versioned API model names like `gpt-4o-mini-2024-07-18`
- Aggregated stats endpoint: total calls, total tokens, total USD cost, and a 10-entry recent usage log

### Background Jobs + Webhook Callbacks (`/api/v1/jobs`)
- Queue email and document processing jobs via REST; jobs accept an optional `callbackUrl`
- Job state persisted to the `job_events` PostgreSQL table — durable history independent of BullMQ's in-memory state
- `GET /jobs/:jobId/status` — poll any job by ID for its DB-persisted status and result
- `GET /jobs` — list all jobs for the current user (most recent 20)
- On job completion, workers update `job_events` and fire an HTTP POST to `callbackUrl` if provided; delivery outcome (`callbackSent`, `callbackSentAt`) recorded on the event

### Observability (`/health`)
- `GET /health/ping` — lightweight liveness check (for load balancers and uptime monitors)
- `GET /health` — deep check: PostgreSQL (`SELECT 1`), Redis (`PING`), system memory (warns at > 90%)
- `GET /health/metrics` — request counts, average/p99 response times, process memory, Node version, PID
- `GET /health/performance` — slow request log, memory snapshots, top endpoints by hit count

### Rate Limiting
- Global limiter: 100 requests / 15 minutes per IP (Redis-backed)
- Auth limiter: 10 requests / 15 minutes per IP on all `/api/v1/auth` routes

---

## API Reference

Interactive Swagger documentation is available at `/api/docs` when the server is running.

### Auth
| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/v1/auth/register` | — | Create a new user account |
| POST | `/api/v1/auth/login` | — | Login — returns `accessToken` + `refreshToken` |
| POST | `/api/v1/auth/refresh` | — | Exchange refresh token for a new access token |
| POST | `/api/v1/auth/logout` | — | Invalidate refresh token and current access token |

### Users
| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/v1/users/me` | Bearer | Current user profile (Redis cache-first, session guard) |
| GET | `/api/v1/users/admin` | Bearer + admin role | Admin-only protected route |

### Organisations
| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/v1/organisations` | Bearer | Create an organisation (caller becomes admin) |
| GET | `/api/v1/organisations/me` | Bearer | Get current user's organisation |
| GET | `/api/v1/organisations/members` | Bearer + admin | List all members in the organisation |
| POST | `/api/v1/organisations/invite` | Bearer + admin | Add an existing user to the organisation by email |

### Billing
| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/v1/billing/plans` | Bearer | Current plan details and available tier pricing |
| POST | `/api/v1/billing/upgrade` | Bearer | Upgrade or downgrade to a different plan |
| GET | `/api/v1/billing/history` | Bearer | Chronological log of billing events |

### AI
| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/v1/ai/analyze` | Bearer | Analyze text — sentiment, key points, topics |
| POST | `/api/v1/ai/summarize` | Bearer | Summarize text with bullet points and reading time |
| POST | `/api/v1/ai/classify` | Bearer | Classify text into caller-provided categories |

### Documents
| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/v1/documents` | Bearer | Create a document |
| GET | `/api/v1/documents` | Bearer | List user's documents |
| GET | `/api/v1/documents/:id` | Bearer | Get a specific document |
| DELETE | `/api/v1/documents/:id` | Bearer | Delete a document |
| POST | `/api/v1/documents/:id/analyze` | Bearer | Run AI analysis (cached after first run) |
| POST | `/api/v1/documents/:id/embed` | Bearer | Generate and store vector embeddings |

### Search and RAG
| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/v1/search` | Bearer | Semantic vector search across documents |
| POST | `/api/v1/rag/ask` | Bearer | Ask a question answered from document corpus (semantic cache-first) |

### Jobs
| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/v1/jobs/email` | Bearer | Queue an email job (optional `callbackUrl`) |
| POST | `/api/v1/jobs/document` | Bearer | Queue a document processing job (optional `callbackUrl`) |
| GET | `/api/v1/jobs` | Bearer | List all jobs for current user (last 20) |
| GET | `/api/v1/jobs/:jobId/status` | Bearer | Poll a specific job's DB-persisted status |
| GET | `/api/v1/jobs/email/:jobId` | Bearer | Poll email job state from BullMQ |

### Usage
| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/v1/usage` | Bearer | Aggregated token and cost usage statistics |

### Health
| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/health/ping` | — | Liveness check for load balancers |
| GET | `/health` | — | Deep health check (DB, Redis, memory) |
| GET | `/health/metrics` | — | Request counts, response times, process info |
| GET | `/health/performance` | — | Slow request log, memory trend, top endpoints |

---

## Challenges Solved

### Redis TTL Precision Bug (token_nbf)
After implementing token invalidation on logout, freshly issued access tokens were being rejected immediately. Root cause: `Date.now()` (milliseconds) was stored as the `nbf` boundary, but the new token's `iat` is in seconds — making `iat * 1000 < Date.now()` always true for the instant of issuance. The fix was to store `newToken.iat * 1000` as the boundary, so the newly issued token always passes the check.

### OpenAI SDK v6 Responses API Migration
The OpenAI Node SDK v6 introduced the Responses API (`openai.responses.create`) as the preferred interface, replacing the legacy `chat.completions.create`. Adapting required updating all call sites, adjusting to `input_tokens`/`output_tokens` field names, and building a `handleAIError` utility that normalises error codes across both API surfaces.

### pgvector with Prisma
Prisma does not natively support binding `vector` types as query parameters. The schema uses `previewFeatures = ["postgresqlExtensions"]` with `extensions = [vector]` in the datasource block — a combination that required careful version alignment with Prisma 6. All vector insertions and cosine similarity queries use the raw `pg` pool with parameterised SQL (`$1::vector`) rather than going through Prisma.

### Structured JSON Reliability from LLMs
Instructing a model to return pure JSON without markdown fences is not always reliable in practice. The `parseAIJson()` utility implements a three-pass parsing strategy: direct `JSON.parse` → markdown fence stripping → regex extraction of the first `{...}` block. This makes AI response parsing robust without requiring the model's structured output mode.

### Security Hardening — Logout and Session Guard
A common oversight in JWT systems is that logout only clears client-side storage, leaving the token valid until expiry. Here, logout writes a `token_nbf` key to Redis that server-side invalidates the current access token immediately. Additionally, `GET /users/me` queries with `NOT: { refreshToken: null }`, so a logged-out user gets a `401` instantly regardless of the token's remaining TTL.

### Idempotent Login with Session Detection
Calling login twice with valid credentials would otherwise create duplicate sessions. The service checks for an active session (`refreshToken != null AND refreshTokenExpiresAt > now`) before issuing new tokens — returning the existing session and a `loginStatus: 2` signal so clients can handle the "already logged in" state without showing duplicate login errors.

### Semantic Cache Vector Index Design
Naively storing one Redis key per cached question does not support similarity lookup — you would need an exact key match. Instead, the semantic cache maintains a per-user *vector index*: a single Redis key holding an array of `{ question, vector, cacheKey }` entries. On each query the index is loaded once, all cosine similarities are computed in-process, and the best match is checked against the threshold. This avoids a separate vector database while keeping RAG response times fast for repeat-intent questions.

### Separating Durable Job State from BullMQ
BullMQ's job state lives in Redis and is ephemeral — jobs are pruned on completion. Clients that need to check job outcomes hours later, or receive webhook callbacks, need a durable record. The `job_events` PostgreSQL table captures every job at creation and is updated by the worker on completion or failure. This lets clients poll `/jobs/:jobId/status` indefinitely and keeps a full audit trail independent of Redis retention settings.

---

## What I Learned

### Node.js Patterns from a Laravel Background
Coming from Laravel's MVC conventions, the biggest mental shift was understanding that Node.js has no built-in structure — you build it deliberately. The Controller → Service pattern here mirrors Laravel's approach but is assembled from first principles: `asyncHandler` wrappers instead of try-catch in every controller, a centralised error middleware that handles all thrown errors uniformly by reading `err.status`, and Joi schemas at the route layer instead of Form Requests.

### AI Prompt Design
Building reliable AI integrations means treating prompts like contracts: specify the exact output schema, enumerate what the model must never do, and validate the output defensively on receipt. The `promptBuilder.js` pattern — returning `{ system, user }` pairs — keeps prompts composable, version-controlled, and testable independently of the HTTP layer.

### Token Economics and Cost Awareness
Tracking per-request token usage revealed how quickly costs accumulate at scale. Storing cost at insertion time (rather than computing it on read) ensures historical accuracy even if pricing changes. The cost calculator normalises versioned model names so pricing stays correct as OpenAI ships new model versions under the same family name.

### Vector Search and RAG Design
Building a RAG system from scratch — rather than using a framework — required understanding each stage independently: what chunking strategy minimises context loss (overlapping windows with paragraph-aware splitting), how cosine similarity translates to a meaningful confidence signal (weighted toward the top result rather than a naive average), and how to write a prompt that keeps the model genuinely grounded in provided sources rather than drifting to its training data.

### Semantic Caching as a Cost Control Mechanism
At scale, many user questions have the same intent even if phrased differently. Storing question embeddings in a lightweight Redis vector index and skipping OpenAI calls when similarity exceeds 0.85 cuts costs and latency for the most common queries. The key design constraint was choosing a threshold high enough to avoid false matches while low enough to catch genuine paraphrases.

### Multi-Tenancy as a Schema Concern, Not an App Concern
Adding `org_id` to every data table makes multi-tenancy a query filter rather than application logic. The `attachOrg` middleware resolves the org once per request, and services receive `orgId` as an explicit argument. This keeps tenant isolation auditable at the data layer and avoids scattering ownership checks throughout the codebase.

---

## Local Setup

### Prerequisites
- Node.js v18+
- PostgreSQL 14+ with the pgvector extension
- Redis 6+
- An OpenAI API key

### Clone and Install

```bash
git clone <repo-url>
cd ai-saas-backend
npm install
npx prisma generate
```

### Environment

Copy `.env.example` to `.env` and fill in your values:

```bash
cp .env.example .env
```

```env
PORT=5000
NODE_ENV=development

DATABASE_URL=postgresql://postgres:postgres@localhost:5432/ai_saas_dev
DB_HOST=localhost
DB_PORT=5432
DB_NAME=ai_saas_dev
DB_USER=postgres
DB_PASSWORD=your_password

JWT_SECRET=your_jwt_secret_here
JWT_EXPIRES_IN=15m
JWT_REFRESH_SECRET=your_refresh_secret_here
JWT_REFRESH_EXPIRES_IN=7d

REDIS_HOST=localhost
REDIS_PORT=6379

OPENAI_API_KEY=your_openai_api_key_here
OPENAI_MODEL=gpt-4o-mini
```

### Database Setup

Enable the pgvector extension in PostgreSQL:

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

Run Prisma migrations:

```bash
npx prisma migrate dev
```

### Run

```bash
# Development (hot reload)
npm run dev

# Production
npm start
```

Server starts on `http://localhost:5000`. Swagger docs: `http://localhost:5000/api/docs`.

### WSL2 / Ubuntu Notes
- Start services before running the app: `sudo service postgresql start && sudo service redis-server start`
- If Redis refuses connections, set `REDIS_HOST=127.0.0.1` instead of `localhost` to avoid IPv6 resolution issues

---

## Project Structure

```
prisma/
  schema.prisma             # Models: User, Organisation, Document, Embedding,
                            #         AiUsage, RagHistory, BillingEvent, JobEvent
src/
  app.js                    # Entry point — middleware, routes, startup sequence
  config/
    env.js                  # Environment variable loader and validator
    db.js                   # Legacy pg Pool (used for /db/test health route only)
    prisma.js               # Prisma client singleton
    redis.js                # Redis client with connection lifecycle management
    rateLimiter.js          # Global (100/15m) and auth (10/15m) limiters
    queue.js                # BullMQ queue instances
    swagger.js              # Swagger/OpenAPI spec configuration
    openai.js               # OpenAI client singleton
    plans.js                # Plan definitions — free/pro/enterprise limits
    metrics.js              # In-process request metrics collector
  routes/                   # Route definitions — no logic
  controllers/              # HTTP parsing, calls service, returns JSON
  services/
    auth.service.js         # Register, login, logout, refresh token logic
    organisation.service.js # Org creation, member management, invite-by-email
    document.service.js     # Document CRUD + AI analysis with result caching
    embedding.service.js    # Document chunking and pgvector storage
    search.service.js       # Cosine similarity search with pgvector
    rag.service.js          # Full RAG pipeline with semantic cache + confidence scoring
    ai.service.js           # Analyze, summarize, classify via OpenAI
    usage.service.js        # Token and USD cost tracking aggregation
    billing.service.js      # Plan upgrade/downgrade + billing event recording
    limit.service.js        # Per-plan daily request, monthly token, document limits
    jobEvent.service.js     # job_events CRUD — durable job state in PostgreSQL
    health.service.js       # DB/Redis/memory checks, metrics, performance summaries
  middlewares/
    auth.middleware.js      # Bearer JWT verification + token_nbf invalidation check
    org.middleware.js       # attachOrg — resolves orgId from DB, sets req.orgId
    planLimit.middleware.js # enforceRequestLimit — checks and increments daily counter
    validate.middleware.js  # Joi request schema validation
    logger.middleware.js    # Per-request method + URL logger
    error.middleware.js     # Centralised error handler — reads err.status
  utils/
    promptBuilder.js        # All system/user prompt builders + parseAIJson
    embeddings.js           # Embedding generation + overlapping chunk algorithm
    semanticCache.js        # Redis vector index — cosine similarity lookup for RAG cache
    costCalculator.js       # Per-model USD cost calculation with name normalisation
    aiErrorHandler.js       # OpenAI error normalisation + withRetry wrapper
    webhook.js              # HTTP webhook delivery with timeout + delivery tracking
    asyncHandler.js         # Async controller error-forwarding wrapper
    token.js                # JWT sign/verify helpers
    cache.js                # Redis get/set/delete/deleteByPattern helpers
    response.js             # sendSuccess() — consistent JSON response envelope
  workers/
    email.worker.js         # BullMQ email job consumer + job_events update + webhook
    document.worker.js      # BullMQ document job consumer + job_events update + webhook
    embedding.worker.js     # BullMQ embedding job consumer
    index.js                # startWorkers() — initialises all workers
```

---

## Author

**KASHIF UMAR**

© 2025 All rights reserved. Unauthorized reproduction is not permitted.
