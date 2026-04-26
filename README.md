# AI SaaS Backend

A production-grade REST API backend for an AI-powered SaaS platform. Built to demonstrate real-world backend engineering skills — from secure JWT authentication and Redis-backed session management to OpenAI integration with cost tracking, vector embeddings, and Retrieval-Augmented Generation (RAG).

This project was built independently as part of a structured learning journey toward becoming an AI-integrated backend architect.

---

## What Problem It Solves

Most AI SaaS Systems stop at "call the OpenAI API and return the result." This project goes further by solving the production concerns that actually matter:

- **Who can call the AI?** — Full JWT auth with refresh token rotation and Redis-based token invalidation
- **How much does it cost per user?** — Per-request token and USD cost tracking stored in PostgreSQL
- **How do you search large documents intelligently?** — pgvector-powered semantic search with overlapping text chunks and cosine similarity scoring
- **How do you answer questions grounded in user documents?** — A full RAG pipeline: embed → store → retrieve → prompt → respond
- **How do you prevent abuse?** — Redis-backed rate limiting at both global and auth-specific levels
- **How do you offload slow work?** — BullMQ background queues for email and document processing jobs

---

## Tech Stack

| Technology | Role | Why |
|---|---|---|
| **Node.js + Express 5** | HTTP server | Lightweight, non-blocking I/O; Express 5 adds native async error propagation |
| **PostgreSQL + pgvector** | Primary database + vector store | Single database for relational data and high-dimensional embedding search |
| **Prisma ORM** | Database access | Type-safe queries, schema-as-code, clean migration workflow |
| **Redis** | Cache + rate limit store | Sub-millisecond access token caching; atomic TTL operations for session invalidation |
| **BullMQ** | Background job queues | Redis-backed, reliable job processing with retry support |
| **OpenAI SDK v6** | AI completions + embeddings | GPT-4o-mini for text operations; `text-embedding-3-small` for 1536-dim semantic vectors |
| **JWT (HS256)** | Auth tokens | Stateless access tokens (15m) + stateful refresh tokens (7d) stored in DB |
| **bcryptjs** | Password hashing | Salted hashing with configurable rounds |
| **Joi** | Request validation | Declarative schema validation at the route layer before any business logic runs |
| **Swagger UI** | API documentation | Auto-generated interactive docs served at `/api/docs` |

---

## Architecture

### Service Layer Pattern

All business logic lives in `src/services/`. Controllers handle only HTTP concerns — parsing the request, calling a service, returning a response. This keeps controllers thin and services independently testable.

```
Request
  → Rate Limiter (Redis-backed)
  → Logger Middleware
  → Auth Middleware (JWT + token_nbf check)
  → Validation Middleware (Joi)
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

### Background Job Design

BullMQ queues (`emailQueue`, `documentQueue`) are defined in `src/config/queue.js`. Workers in `src/workers/` consume these queues asynchronously. Workers are started only after the PostgreSQL and Redis connections are verified — ensuring no jobs run against an unready system.

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
    → generateEmbedding() converts question to a 1536-dim vector
    → pgvector cosine similarity search (<=> operator) retrieves top-k chunks
    → Confidence scored: 70% top-chunk similarity + 30% average similarity
    → buildRAGPrompt() injects chunks as numbered sources with relevance scores
    → OpenAI generates a grounded answer (temperature 0.1 for factual accuracy)
    → Q&A pair saved to rag_history for auditability
    → Token usage tracked and USD cost calculated per request
```

---

## Key Features and Modules

### Authentication (`/api/v1/auth`)
- Register, login, logout, and token refresh
- Refresh token rotation with database persistence and expiry enforcement
- Redis-based access token invalidation on logout (token_nbf pattern)
- Idempotent login — detects active sessions and returns existing tokens
- `GET /users/me` session guard: returns `401` if the user has logged out, even if the JWT has not yet expired

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
- Confidence levels: `high` / `medium` / `low` / `none` based on similarity score distribution
- Response includes answer, confidence, source citations with relevance scores, and token usage
- Q&A history persisted for audit trail

### Cost-Aware Usage Tracking (`/api/v1/usage`)
- Every OpenAI call records prompt tokens, completion tokens, total tokens, and calculated USD cost
- Per-model pricing table with base-model normalisation — handles versioned API model names like `gpt-4o-mini-2024-07-18`
- Aggregated stats endpoint: total calls, total tokens, total USD cost, and a 10-entry recent usage log

### Background Jobs (`/api/v1/jobs`)
- Queue email and document processing jobs via REST
- Poll job status by ID
- BullMQ workers consume jobs independently of the HTTP request lifecycle

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
| POST | `/api/v1/rag/ask` | Bearer | Ask a question answered from document corpus |

### Jobs and Usage
| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/v1/jobs/email` | Bearer | Queue an email job |
| POST | `/api/v1/jobs/document` | Bearer | Queue a document processing job |
| GET | `/api/v1/jobs/email/:jobId` | Bearer | Poll email job status |
| GET | `/api/v1/usage` | Bearer | Get aggregated token and cost usage statistics |

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
  schema.prisma             # Models: User, Document, Embedding, AiUsage
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
  routes/                   # Route definitions — no logic
  controllers/              # HTTP parsing, calls service, returns JSON
  services/
    auth.service.js         # Register, login, logout, refresh token logic
    document.service.js     # Document CRUD + AI analysis with result caching
    embedding.service.js    # Document chunking and pgvector storage
    search.service.js       # Cosine similarity search with pgvector
    rag.service.js          # Full RAG pipeline with confidence scoring
    ai.service.js           # Analyze, summarize, classify via OpenAI
    usage.service.js        # Token and USD cost tracking aggregation
  middlewares/
    auth.middleware.js      # Bearer JWT verification + token_nbf invalidation check
    validate.middleware.js  # Joi request schema validation
    logger.middleware.js    # Per-request method + URL logger
    error.middleware.js     # Centralised error handler — reads err.status
  utils/
    promptBuilder.js        # All system/user prompt builders + parseAIJson
    embeddings.js           # Embedding generation + overlapping chunk algorithm
    costCalculator.js       # Per-model USD cost calculation with name normalisation
    aiErrorHandler.js       # OpenAI error normalisation + withRetry wrapper
    asyncHandler.js         # Async controller error-forwarding wrapper
    token.js                # JWT sign/verify helpers
    cache.js                # Redis get/set/delete/deleteByPattern helpers
    response.js             # sendSuccess() — consistent JSON response envelope
  workers/
    email.worker.js         # BullMQ email job consumer
    document.worker.js      # BullMQ document job consumer
    index.js                # startWorkers() — initialises all workers
```

---

## Author

**KASHIF UMAR**

[LinkedIn](https://www.linkedin.com/in/kashif-umar/) · [X](https://x.com/kashif_umar)

© 2025 All rights reserved. Unauthorized reproduction is not permitted.
