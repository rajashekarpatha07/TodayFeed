# AI News Platform — Complete Production Build Specification

**Version 1.0 — Backend & Agent Pipeline Reference**
**Status:** Pre-implementation specification. Frontend is out of scope; this document exists so the backend can be built end-to-end and then handed to a frontend developer against a stable, documented API contract.

---

## Table of Contents

1. Executive Summary
2. Product Scope & Non-Goals
3. System Architecture
4. Tech Stack (Full, with Rationale)
5. Authentication & Authorization (Clerk)
6. Database Design (Complete Schema)
7. Data Collection Layer (Collectors)
8. Deduplication & Data Integrity
9. Agent Pipeline (Mastra) — Full Design
10. Job Queue, Scheduling & Concurrency
11. Delivery System (Email + In-App)
12. REST API Specification (Full Contract)
13. Security
14. Error Handling & Logging
15. Testing Strategy
16. Monitoring & Observability
17. Deployment Architecture (AWS EC2)
18. Project & Folder Structure
19. Environment Variables (Complete Reference)
20. Development Roadmap (Detailed)
21. Startup Intelligence Module (Flagship Feature Spec)
22. Newsletter & Web Content Structure
23. Future Roadmap (V2+)
24. Key Architectural Decisions Log
25. Glossary

---

## 1. Executive Summary

This platform is a multi-user, production-grade news intelligence system. Users sign up (via Clerk), choose the categories they care about (AI, startups, space, sports, geopolitics, etc.), and receive a curated, LLM-summarized briefing twice a day — by email, in the web app, or both.

Under the hood, a scheduled pipeline collects raw content from news sites, GitHub, Hacker News, Reddit, and arXiv; deduplicates it against everything previously seen; runs it through a chain of specialized AI agents (filter → rank → summarize → analyze trends → write); stores the result once per run; and then renders a personalized version for every user based on their preferences.

The backend is built to be **handed off cleanly**: a separate frontend developer should be able to build the web app against nothing but the OpenAPI specification this backend produces, without needing to understand the agent pipeline internals.

---

## 2. Product Scope & Non-Goals

**In scope for this document:**
- Backend API (Fastify, REST, OpenAPI)
- Authentication integration (Clerk, server-side verification only)
- Database schema and data lifecycle
- Data collection from external sources
- Agent pipeline (Mastra) for filtering, ranking, summarizing, trend analysis, newsletter generation
- Scheduling and job processing
- Email delivery and in-app briefing storage
- Deployment to a single AWS EC2 instance

**Explicitly out of scope for now:**
- Frontend UI/UX (handed off separately once the API is stable)
- Payments/billing
- Multi-region or multi-instance scaling (documented as a future step only)
- Mobile apps

---

## 3. System Architecture

### 3.1 Component Diagram

```text
                                    ┌───────────────────────────┐
                                    │          Web App           │
                                    │   (separate repo, later)   │
                                    └─────────────┬───────────────┘
                                                  │ HTTPS, Bearer <Clerk session JWT>
                                                  ▼
                        ┌───────────────────────────────────────────────┐
                        │              Backend API (Fastify)              │
                        │  - Clerk auth guard (all routes except /health) │
                        │  - Zod request/response validation              │
                        │  - OpenAPI 3 spec auto-generated                 │
                        │  - Rate limiting, CORS                          │
                        └───────┬───────────────────────────┬─────────────┘
                                │                             │
                    reads/writes │                             │ enqueue / status
                                ▼                             ▼
                     ┌───────────────────┐          ┌──────────────────────┐
                     │     PostgreSQL      │          │   Redis (BullMQ)      │
                     │  (Prisma-managed)   │          │  briefing-run queue   │
                     └───────────────────┘          └──────────┬─────────────┘
                                                                  │
                                                                  ▼
                                                  ┌────────────────────────────┐
                                                  │   Briefing Worker Process    │
                                                  │  (separate PM2 process,      │
                                                  │   consumes queue jobs)       │
                                                  └──────────────┬─────────────┘
                                                                  │
                                                                  ▼
                                                  ┌────────────────────────────┐
                                                  │      Mastra Workflow         │
                                                  │  collect → dedupe → filter   │
                                                  │  → rank → summarize → trend  │
                                                  │  → write newsletter          │
                                                  └───────┬─────────┬──────────┘
                                                          │         │
                                     ┌────────────────────┘         └───────────────┐
                                     ▼                                              ▼
                     ┌───────────────────────────┐                  ┌───────────────────────────┐
                     │   Collector Tools           │                  │   Groq (via Mastra model    │
                     │   Firecrawl / GitHub API /  │                  │   router: "groq/<model>")   │
                     │   HN Algolia / Reddit /     │                  └───────────────────────────┘
                     │   arXiv                     │
                     └───────────────────────────┘
                                                                  │
                                                                  ▼
                                          Persist BriefingRun + BriefingItems (Postgres)
                                                                  │
                                    ┌─────────────────────────────┴─────────────────────────────┐
                                    ▼                                                             ▼
                     For each subscribed User:                                     DeliveryLog written
                     build personalized UserBriefing                               per user, per channel
                                    │
                        ┌───────────┴────────────┐
                        ▼                         ▼
                  Brevo (email)          Available immediately via
                  if "email" in          GET /briefings (in-app)
                  channels               if "inapp" in channels
```

### 3.2 Two Core Flows

**Flow A — Scheduled/Manual Briefing Run (backend-internal, no user involved)**
1. Cron (`0 8 * * *` / `0 20 * * *`) or an admin API call enqueues a `run-briefing` job on BullMQ.
2. The Briefing Worker picks up the job, sets a new `BriefingRun` row to `RUNNING`.
3. The Mastra workflow executes: collect → dedupe → filter → rank → summarize → analyze trends → write.
4. Results are persisted as one `BriefingRun` plus its `BriefingItem` rows. Status becomes `COMPLETED` (or `FAILED`, with `errorMessage` set and the job retried per BullMQ policy).
5. For every `User` with an active `UserPreference`, the worker builds a `UserBriefing` (filtered to their `categorySlugs`, rendered as personalized Markdown).
6. Delivery: if the user's `deliveryChannels` includes `email`, Brevo sends it; a `DeliveryLog` row is written either way (`sent`/`failed`/`skipped`).

**Flow B — User Views Their Briefings (via the future web app)**
1. Web app authenticates the user via Clerk, attaches the session JWT to `GET /api/v1/briefings`.
2. Backend verifies the JWT, resolves the local `User` row (creating it on first request if needed), and returns that user's `UserBriefing` list, most recent first.
3. `GET /api/v1/briefings/:id` returns the full personalized Markdown/content for one entry.
4. `POST /api/v1/briefings/:id/read` marks it read.

---

## 4. Tech Stack (Full, with Rationale)

| Layer | Choice | Package(s) | Why |
|---|---|---|---|
| Runtime | Node.js 22 LTS | — | AWS-friendly, long support window |
| Language | TypeScript | `typescript` | Type safety across many external integrations |
| Web framework | Fastify | `fastify` | Faster than Express, native async support, strong plugin ecosystem for OpenAPI |
| Auth | Clerk (server-side verification) | `@clerk/backend` | Hosted auth; backend only verifies JWTs, never touches passwords |
| ORM | Prisma | `@prisma/client`, `prisma` | Type-safe queries, migration tooling |
| Database | PostgreSQL 16 | — | Relational integrity for multi-tenant data |
| Queue | BullMQ + Redis | `bullmq`, `ioredis` | Reliable retries/backoff for scheduled + on-demand runs |
| Agent framework | Mastra | `@mastra/core`, `mastra` (CLI, dev-only) | TypeScript-native agents/tools/workflows; no CrewAI equivalent exists in JS/TS |
| LLM provider | Groq | (via Mastra's model router, no separate SDK needed) | Fast inference; models referenced as `"groq/<model-id>"` |
| Web crawling | Firecrawl | `@mendable/firecrawl-js` | Wrapped as a Mastra tool for the Research agent |
| Email | Brevo | `@getbrevo/brevo` | Sending + deliverability, already in use |
| API docs | Swagger/OpenAPI | `@fastify/swagger`, `@fastify/swagger-ui` | Auto-generated from route schemas — the frontend contract |
| Validation | Zod | `zod` | Env config, route schemas, Mastra tool I/O schemas |
| Logging | Pino | `pino`, `pino-pretty` | Structured JSON logs in production, readable in dev |
| Rate limiting | `@fastify/rate-limit` | | Protects the public API from abuse |
| CORS | `@fastify/cors` | | Configured to allow only the known frontend origin(s) |
| Testing | Vitest | `vitest` | Fast unit/integration tests |
| Process management | PM2 | — | Runs API and worker as separate managed processes on EC2 |
| Containerization (partial) | Docker Compose | — | Runs only stateful services (Postgres, Redis); app processes run natively via PM2 |

---

## 5. Authentication & Authorization (Clerk)

### 5.1 Flow
1. The (future) frontend integrates Clerk's own sign-in/sign-up components and session management — this backend has no involvement in that UI.
2. Every API request from the frontend carries `Authorization: Bearer <clerk-session-token>`.
3. A Fastify `onRequest` hook (implemented as a plugin) calls `@clerk/backend`'s token verification against `CLERK_SECRET_KEY`. Invalid/expired tokens → `401 Unauthorized`.
4. On success, the hook attaches `request.clerkUserId` to the request context.
5. A second step (`ensureLocalUser`) looks up `User.clerkId === request.clerkUserId`; if none exists, it creates one (using the email claim from the verified token). This keeps user provisioning implicit — no separate "register" endpoint is needed.

### 5.2 Authorization Rules
| Rule | Enforcement |
|---|---|
| A user can only read/write their own `UserPreference` and `UserBriefing` rows | Every query includes `where: { userId: request.localUserId }` |
| Admin-only routes (e.g. manual run trigger) | Checked against `User.isAdmin`, itself set only for Clerk user IDs listed in `CLERK_ADMIN_USER_IDS` at startup/seed time |
| Unauthenticated access | Only `GET /health` is exempt from the auth hook |

### 5.3 Session Lifecycle Notes
- Token expiry/refresh is entirely handled by Clerk on the frontend; the backend simply rejects expired tokens with `401`, and the frontend is responsible for refreshing and retrying.
- No server-side session store is needed — this is a stateless verification model.

---

## 6. Database Design (Complete Schema)

### 6.1 Entity-Relationship Overview

```text
User ──1:1── UserPreference
User ──1:N── UserBriefing ──N:1── BriefingRun ──1:N── BriefingItem ──N:1── Article
User ──1:N── DeliveryLog
Category ──1:N── Source
```

### 6.2 Full Field Reference

**`User`**
| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | String (cuid) | PK | |
| clerkId | String | unique | Link to Clerk identity |
| email | String | unique | Synced from Clerk token claims |
| name | String? | nullable | |
| isAdmin | Boolean | default false | |
| createdAt / updatedAt | DateTime | auto | |

**`UserPreference`**
| Field | Type | Constraints | Notes |
|---|---|---|---|
| id | String (cuid) | PK | |
| userId | String | unique, FK → User | 1:1 |
| categorySlugs | String[] | default `[]` | e.g. `["ai","startups","space"]` |
| deliveryChannels | String[] | default `["email"]` | `"email"`, `"inapp"` |
| deliveryTimes | String[] | default `["08:00","20:00"]` | 24h, user's local timezone |
| timezone | String | default `"Asia/Kolkata"` | IANA timezone string |

**`Category`**
| Field | Type | Notes |
|---|---|---|
| id | String (cuid) | PK |
| slug | String, unique | e.g. `"ai"` |
| label | String | e.g. `"Artificial Intelligence"` |
| emoji | String? | for UI display |

**`Source`**
| Field | Type | Notes |
|---|---|---|
| id | String (cuid) | PK |
| name | String | e.g. `"TechCrunch"` |
| url | String | base URL / feed |
| type | Enum `SourceType` | `FIRECRAWL`, `GITHUB_API`, `HACKERNEWS_API`, `REDDIT_API`, `ARXIV_API` |
| categoryId | String, FK → Category | |
| isActive | Boolean, default true | allows disabling a noisy source without deleting history |

**`Article`**
| Field | Type | Notes |
|---|---|---|
| id | String (cuid) | PK |
| url | String, unique | |
| urlHash | String, unique | SHA-256 of normalized URL — **the** dedup key |
| title | String | |
| content | Text? | raw extracted content |
| categorySlug | String | denormalized for fast filtering |
| sourceName | String | denormalized |
| publishedAt | DateTime? | |
| createdAt | DateTime | when we collected it |
| Indexes | | `categorySlug`, `createdAt` |

**`BriefingRun`**
| Field | Type | Notes |
|---|---|---|
| id | String (cuid) | PK |
| slot | Enum `RunSlot` | `MORNING`, `EVENING`, `MANUAL` |
| status | Enum `RunStatus` | `PENDING`, `RUNNING`, `COMPLETED`, `FAILED` |
| startedAt / completedAt | DateTime | |
| errorMessage | String? | populated on failure |
| trendNotes | Text? | output of the Trend Analysis agent, shared across all users |

**`BriefingItem`**
| Field | Type | Notes |
|---|---|---|
| id | String (cuid) | PK |
| briefingRunId | String, FK → BriefingRun | |
| articleId | String, FK → Article | |
| categorySlug | String | |
| rankScore | Int | 0–10 scale from the Ranking agent |
| summary | Text | |
| whyItMatters | Text | |
| Indexes | | `briefingRunId`, `categorySlug` |

**`UserBriefing`**
| Field | Type | Notes |
|---|---|---|
| id | String (cuid) | PK |
| userId | String, FK → User | |
| briefingRunId | String, FK → BriefingRun | |
| personalizedMd | Text | rendered Markdown, filtered to the user's categories |
| isRead | Boolean, default false | |
| createdAt | DateTime | |
| Constraint | | unique `(userId, briefingRunId)` — never duplicated |
| Indexes | | `(userId, createdAt)` |

**`DeliveryLog`**
| Field | Type | Notes |
|---|---|---|
| id | String (cuid) | PK |
| userId | String, FK → User | |
| channel | String | `"email"` \| `"inapp"` |
| status | String | `"sent"` \| `"failed"` \| `"skipped"` |
| error | String? | |
| sentAt | DateTime | |
| Indexes | | `userId` |

### 6.3 Migration Discipline
- All schema changes go through Prisma migrations (`prisma migrate dev` locally, `prisma migrate deploy` in production) — never manual SQL against the production database.
- Seed data (`scripts/seed.ts`) populates `Category` and `Source` rows on first setup, and can optionally create a test user/preference for local development.

---

## 7. Data Collection Layer (Collectors)

Each collector is wrapped as a **Mastra Tool** (typed input/output via Zod) so the Research Agent can call it directly, but the underlying fetch logic is plain, deterministic TypeScript — the LLM does not decide *how* to scrape, only *what* to ask for.

| Collector | Method | Detail |
|---|---|---|
| News sites | Firecrawl | Crawl → extract → return clean Markdown per URL. Configured per `Source` row with `type = FIRECRAWL`. |
| GitHub | GitHub REST API | Trending repos (via search API sorted by stars/forks in a date window), releases feed, issue/PR counts. Optional `GITHUB_TOKEN` raises rate limits from 60/hr to 5,000/hr. |
| Hacker News | Algolia HN Search API | Top stories and new discussions, filtered by score threshold and time window. |
| Reddit | Reddit JSON endpoints (`/r/<sub>/top.json`) | Pulls from a fixed subreddit list (see §7.1); no auth required for read-only JSON endpoints, though `REDDIT_CLIENT_ID`/`SECRET` allow the authenticated API for higher limits. |
| arXiv | arXiv API (Atom feed) | Recent papers per category (`cs.AI`, `cs.CL`, etc.), parsed for title/abstract/authors/link. |

### 7.1 Subreddit List
`r/programming`, `r/OpenSource`, `r/MachineLearning`, `r/javascript`, `r/node`, `r/startups`, `r/technology`

### 7.2 Collection Window
Every run (morning or evening) collects content published since the **previous successful run's `completedAt`**, not a fixed "12 hours," so a delayed or failed run doesn't create a gap — the next run simply covers a longer window.

---

## 8. Deduplication & Data Integrity

Deduplication happens at **two layers**, deliberately redundant:

1. **Collection-time:** before an `Article` row is inserted, its `urlHash` (SHA-256 of the normalized URL) is checked against existing rows. If found, the existing row is reused rather than re-inserted or re-summarized.
2. **Delivery-time:** the `UserBriefing` unique constraint on `(userId, briefingRunId)` guarantees a user is never sent the same run twice, even if a delivery job is retried after a partial failure.

**URL normalization rule** (applied before hashing): lowercase, trim whitespace, strip trailing slashes. Query-string tracking parameters (`utm_*`, `ref`, etc.) should also be stripped during normalization to avoid treating the same article as new just because of a different campaign tag.

---

## 9. Agent Pipeline (Mastra) — Full Design

### 9.1 Primitives Used
- **Tool** — a typed function (Zod input/output schema) an agent can call. All five collectors are tools.
- **Agent** — an LLM (via Mastra's model router) + system instructions + optional tools.
- **Workflow** — a typed, multi-step chain (`.then()`, `.branch()`, `.parallel()`) combining agents and plain deterministic functions (like the dedupe check, which is *not* an agent — it's a direct database query).

### 9.2 Agent-by-Agent Specification

**1. Research Agent**
- **Tools:** all five collectors.
- **Instructions (summary):** for each active `Category`/`Source`, fetch content published since the last run; return a normalized array of candidate articles.
- **Output schema:** `{ title, url, category, publishedAt, content, sourceName }[]`

**2. Filter Agent**
- **Not a summarizer** — a gatekeeper.
- **Instructions (summary):** discard clickbait, near-duplicate coverage of the same underlying story, advertisements/sponsored content, unverified rumors. Retain: announcements, product launches, security issues, research papers, OSS releases, government/policy decisions, major sports results, space missions, and (per §21) structured startup funding news.
- **Output schema:** same shape as input, minus discarded items, plus a `keepReason` string per item (useful for debugging/QA, not shown to end users).

**3. Ranking Agent**
- **Instructions (summary):** score each surviving item 0–10 on importance, relevance, and freshness; return only the top 5–8 per category, sorted descending.
- **Output schema:** adds `rankScore: number` per item.

**4. Summarization Agent**
- **Instructions (summary):** for each item, produce `summary`, `whyItMatters`. Style: concise, factual, no editorializing beyond stated "why it matters."
- **Output schema:** adds `summary`, `whyItMatters` per item — this is the shape persisted as `BriefingItem`.

**5. Trend Analysis Agent**
- **Runs over the full ranked/summarized set at once**, not per-article.
- **Instructions (summary):** identify cross-cutting patterns — repeated themes, sector momentum (especially startup funding, per §21), notable absences. Output free-text `trendNotes`, persisted directly on `BriefingRun`.

**6. Newsletter Agent**
- **Instructions (summary):** assemble all `BriefingItem`s (grouped by category, in rank order) plus `trendNotes` into one shared Markdown document following the structure in §22. This shared document is the *superset*; per-user personalization (§9.3) is a deterministic filter over it, not a separate LLM call.

### 9.3 Personalization (Deterministic, Not an Agent)
Per-user `UserBriefing.personalizedMd` is generated by filtering the shared Newsletter Agent output down to the categories in that user's `UserPreference.categorySlugs`, then re-rendering the section headers/order. This step is plain TypeScript, not an LLM call — it's a rendering/filtering operation, not a judgment call, so it doesn't need to be an agent.

### 9.4 Error Handling Within the Workflow
- Each agent step validates its output against its Zod schema before the workflow proceeds; a schema failure fails the step (with the raw model output logged) rather than silently passing malformed data downstream.
- The workflow itself is invoked from within a BullMQ job, so a failure at any step marks the `BriefingRun` `FAILED` with `errorMessage`, and the job is retried per the queue's backoff policy (see §10).

---

## 10. Job Queue, Scheduling & Concurrency

| Job | Trigger | Queue | Concurrency | Retry policy |
|---|---|---|---|---|
| `run-briefing` | Cron `0 8 * * *`, `0 20 * * *`, or admin API call | `briefing-runs` (BullMQ) | 1 (only one run active at a time — the pipeline is sequential by nature) | 3 attempts, exponential backoff starting at 30s |
| `deliver-user-briefing` | Enqueued once per subscribed user after a `BriefingRun` completes | `deliveries` (BullMQ) | 5 (parallel per-user delivery) | 3 attempts, exponential backoff starting at 10s |

- **node-cron** triggers the two scheduled enqueues; it does not run the pipeline itself — its only job is to push a message onto BullMQ, keeping the actual work fully queue-managed and retryable.
- The **admin trigger** endpoint (`POST /api/v1/admin/runs/trigger`) enqueues the same `run-briefing` job with `slot = MANUAL`, so manual and scheduled runs share identical logic.
- Jobs are idempotent with respect to `Article` dedup — re-running does not re-summarize already-processed articles from the same window.

---

## 11. Delivery System

### 11.1 Email (Brevo)
- Rendered from `UserBriefing.personalizedMd` → HTML (via a lightweight Markdown-to-HTML step; MJML can be introduced later if richer responsive templates are needed).
- Sent via Brevo's transactional email API using `BREVO_SENDER_EMAIL` / `BREVO_SENDER_NAME`.
- Every send attempt writes a `DeliveryLog` row regardless of outcome.

### 11.2 In-App
- No separate delivery step is needed — the moment a `UserBriefing` row exists, it's immediately retrievable via `GET /api/v1/briefings`. A `DeliveryLog` row with `channel = "inapp"`, `status = "sent"` is still written for audit consistency.

### 11.3 Channel Selection
Both channels are independent per user (`UserPreference.deliveryChannels`); a user can have email only, in-app only, or both. Users with **neither** channel selected still get a `UserBriefing` row created (so switching preferences later doesn't lose history) but no delivery attempt — logged as `status = "skipped"`.

---

## 12. REST API Specification (Full Contract)

Base path: `API_BASE_PATH` (default `/api/v1`). All routes require `Authorization: Bearer <clerk-session-jwt>` except `/health`.

### `GET /health`
- **Auth:** none
- **Response 200:** `{ "status": "ok" }`

### `GET /me`
- **Response 200:**
```json
{ "id": "usr_...", "email": "user@example.com", "isAdmin": false }
```
- Creates the local `User` row on first call if it doesn't exist yet.

### `GET /preferences`
- **Response 200:**
```json
{
  "categorySlugs": ["ai", "startups", "space"],
  "deliveryChannels": ["email", "inapp"],
  "deliveryTimes": ["08:00", "20:00"],
  "timezone": "Asia/Kolkata"
}
```

### `PUT /preferences`
- **Request body:** same shape as the response above (all fields optional; partial updates merge with existing values).
- **Response 200:** updated preference object.
- **Response 400:** validation error (e.g. unknown category slug).

### `GET /categories`
- **Response 200:** `[{ "slug": "ai", "label": "Artificial Intelligence", "emoji": "🤖" }, ...]`

### `GET /briefings`
- **Query params:** `?page=1&pageSize=20`
- **Response 200:**
```json
{
  "items": [
    { "id": "brf_...", "createdAt": "...", "isRead": false, "slot": "MORNING" }
  ],
  "page": 1,
  "pageSize": 20,
  "total": 42
}
```

### `GET /briefings/:id`
- **Response 200:** `{ "id": "brf_...", "personalizedMd": "...", "isRead": false, "createdAt": "..." }`
- **Response 404:** briefing does not exist or does not belong to the requesting user.

### `POST /briefings/:id/read`
- **Response 200:** `{ "id": "brf_...", "isRead": true }`

### `POST /admin/runs/trigger`
- **Auth:** requires `User.isAdmin === true`, else `403 Forbidden`.
- **Response 202:** `{ "jobId": "...", "status": "queued" }`

### Standard Error Shape (all routes)
```json
{ "error": { "code": "UNAUTHORIZED", "message": "..." } }
```
Common codes: `UNAUTHORIZED` (401), `FORBIDDEN` (403), `NOT_FOUND` (404), `VALIDATION_ERROR` (400), `INTERNAL_ERROR` (500).

The complete, always-current version of this contract is the auto-generated OpenAPI 3 document served at `/documentation` — this section is a human-readable summary of it, not a replacement.

---

## 13. Security

| Concern | Mitigation |
|---|---|
| Auth bypass | Every route except `/health` runs through the Clerk verification hook; enforced at the plugin level, not per-route, so a forgotten route can't accidentally skip auth |
| Cross-user data access | Every query scoped to `request.localUserId`; no route accepts an arbitrary `userId` from the client |
| Abuse / scraping of the API | `@fastify/rate-limit` applied globally, tighter limits on `/admin/*` |
| CORS | `@fastify/cors` restricted to the known frontend origin(s) once the frontend domain is known; wildcard only in local development |
| Secrets | All API keys (Clerk, Groq, Firecrawl, Brevo, DB, Redis) via environment variables, never committed; `.env.example` documents required keys without values |
| SQL injection | Not applicable directly — Prisma parameterizes all queries |
| LLM output trust | Agent outputs are schema-validated (Zod) before persistence; nothing from an LLM response is executed or interpolated into SQL/HTML unsanitized |
| Admin escalation | `isAdmin` is only ever set via the `CLERK_ADMIN_USER_IDS` seed list at deploy time — no API route can promote a user to admin |

---

## 14. Error Handling & Logging

- **Structured logging** via Pino everywhere; every log line includes a request ID (API) or job ID (worker) for traceability across the two processes.
- **API errors** are normalized to the standard error shape (§12) by a global Fastify error handler — no raw stack traces returned to clients in production.
- **Pipeline errors** (agent schema failures, tool/network failures) are caught per-step, logged with full context (which step, which run, which category), and surface as `BriefingRun.status = FAILED` with a human-readable `errorMessage` — never a silent partial run.
- **Delivery errors** (Brevo failures, etc.) never fail the whole run — they're isolated to that user's `DeliveryLog` row (`status = "failed"`, `error` populated) so one bad email address doesn't block delivery to everyone else.

---

## 15. Testing Strategy

| Layer | Approach |
|---|---|
| Unit | Vitest for pure functions: URL normalization/hashing, preference-merging logic, Markdown personalization filter |
| Integration | Spin up a test Postgres (Docker) + run Prisma migrations; test route handlers against it directly, mocking Clerk verification |
| Agent/tool tests | Mock the LLM calls (Mastra supports test/mock model providers) to assert workflow wiring and schema validation without burning real API calls |
| End-to-end (manual, pre-launch) | Trigger a `MANUAL` run against a small set of test sources and verify the full path: collection → dedupe → pipeline → persistence → delivery to a test user |

---

## 16. Monitoring & Observability

**For v1 (launch), keep this simple and inspectable:**
- Pino logs, shipped to files, rotated and readable via PM2's log commands (`pm2 logs`).
- A lightweight `BriefingRun` status check (`SELECT * FROM "BriefingRun" ORDER BY "startedAt" DESC LIMIT 5`) is enough to see pipeline health at a glance.
- `DeliveryLog` failure counts per day are the primary delivery-health signal.

**Deferred to later (once user count justifies it):**
- Prometheus + Grafana dashboards for queue depth, job duration, API latency.
- Alerting (e.g. on repeated `BriefingRun` failures) via a simple webhook/email, not a full incident-management stack.

---

## 17. Deployment Architecture (AWS EC2)

- **Single EC2 instance** (Ubuntu), sized for a small user base initially — vertical scaling is the first lever if needed, not premature horizontal scaling.
- **Docker Compose** runs exactly two services: `postgres` and `redis` — stateful, easy to back up/reset independently of app deploys.
- **PM2** runs two Node processes directly on the host (not containerized), managed via an ecosystem file:
  - `api` — the Fastify server
  - `worker` — the BullMQ consumer running the Mastra pipeline
- **Deploy flow:** pull latest code → `npm ci` → `npm run build` → `prisma migrate deploy` → `pm2 reload ecosystem.config.js` (zero-downtime reload for the API process; the worker can restart between runs without user-facing impact).
- **Backups:** scheduled `pg_dump` of the Postgres container's volume, stored off-instance (e.g. S3), on a daily cadence at minimum.
- **TLS/reverse proxy:** Nginx (or Caddy) in front of the Fastify API for TLS termination and to serve as the public entry point on port 443.

---

## 18. Project & Folder Structure

```text
ai-news-backend/
  prisma/
    schema.prisma
    migrations/
  src/
    config/
      env.ts                  # Zod-validated environment config
    plugins/
      clerk.ts                # auth verification + ensureLocalUser
      swagger.ts               # OpenAPI generation
    routes/
      v1/
        health.routes.ts
        me.routes.ts
        preferences.routes.ts
        categories.routes.ts
        briefings.routes.ts
        admin.routes.ts
    services/
      db.ts                   # Prisma client singleton
      queue.ts                # BullMQ queue definitions
      email/
        brevo.ts
      markdown/
        personalize.ts        # deterministic per-user filter/render
    mastra/
      index.ts                # Mastra instance registration
      tools/
        firecrawl.tool.ts
        github.tool.ts
        hackernews.tool.ts
        reddit.tool.ts
        arxiv.tool.ts
      agents/
        research.agent.ts
        filter.agent.ts
        ranking.agent.ts
        summarizer.agent.ts
        trend.agent.ts
        newsletter.agent.ts
      workflows/
        briefing.workflow.ts
    jobs/
      scheduler.ts             # node-cron → enqueue
      briefing.worker.ts       # BullMQ consumer, runs the workflow
      delivery.worker.ts        # BullMQ consumer, per-user delivery
    utils/
      logger.ts
      hash.ts
    types/
    app.ts                     # Fastify app assembly
    server.ts                   # entrypoint (API process)
  scripts/
    seed.ts                    # seed Category/Source rows
  docker-compose.yml            # postgres + redis only
  ecosystem.config.js           # PM2 process definitions (api, worker)
  .env.example
  package.json
  tsconfig.json
  README.md
```

---

## 19. Environment Variables (Complete Reference)

| Variable | Required | Example | Purpose |
|---|---|---|---|
| `NODE_ENV` | yes | `production` | Runtime mode |
| `PORT` | yes | `4000` | API listen port |
| `API_BASE_PATH` | yes | `/api/v1` | API route prefix |
| `LOG_LEVEL` | yes | `info` | Pino log level |
| `CLERK_SECRET_KEY` | yes | `sk_live_...` | Server-side token verification |
| `CLERK_PUBLISHABLE_KEY` | yes | `pk_live_...` | Included for parity/documentation; frontend actually uses this |
| `CLERK_ADMIN_USER_IDS` | yes | `user_abc,user_def` | Comma-separated Clerk IDs granted admin |
| `DATABASE_URL` | yes | `postgresql://...` | Prisma connection string |
| `REDIS_URL` | yes | `redis://localhost:6379` | BullMQ connection |
| `GROQ_API_KEY` | yes | `gsk_...` | Auto-detected by Mastra's model router for `groq/*` models |
| `MASTRA_MODEL_RESEARCH` | yes | `groq/llama-3.3-70b-versatile` | Model per agent |
| `MASTRA_MODEL_FILTER` | yes | `groq/deepseek-r1-distill-llama-70b` | |
| `MASTRA_MODEL_RANK` | yes | `groq/llama-3.3-70b-versatile` | |
| `MASTRA_MODEL_SUMMARY` | yes | `groq/llama-3.3-70b-versatile` | |
| `MASTRA_MODEL_TREND` | yes | `groq/deepseek-r1-distill-llama-70b` | |
| `MASTRA_MODEL_NEWSLETTER` | yes | `groq/llama-3.3-70b-versatile` | |
| `FIRECRAWL_API_KEY` | yes | `fc-...` | Firecrawl tool |
| `BREVO_API_KEY` | yes | `xkeysib-...` | Email delivery |
| `BREVO_SENDER_EMAIL` | yes | `briefing@yourdomain.com` | From address |
| `BREVO_SENDER_NAME` | yes | `AI Daily Brief` | From name |
| `GITHUB_TOKEN` | optional | — | Raises GitHub API rate limit |
| `REDDIT_CLIENT_ID` / `REDDIT_CLIENT_SECRET` | optional | — | Authenticated Reddit API access |

---

## 20. Development Roadmap (Detailed)

### Phase 1 — Foundation
- [ ] Fastify app skeleton, `/health` route
- [ ] Clerk verification plugin + `ensureLocalUser`
- [ ] Prisma schema (§6) + initial migration
- [ ] Docker Compose for Postgres/Redis; confirm local connectivity
- [ ] PM2 ecosystem file with `api` and `worker` process stubs

### Phase 2 — Core API
- [ ] `/me`, `/preferences` (GET/PUT), `/categories`
- [ ] OpenAPI generation wired up, spot-checked at `/documentation`
- [ ] Seed script for `Category`/`Source` rows

### Phase 3 — Collection
- [ ] Implement all five collector tools with real API calls
- [ ] URL normalization + hashing + dedup-on-insert logic
- [ ] Manual smoke test: run each collector standalone, confirm output shape

### Phase 4 — Agent Pipeline
- [ ] Register Mastra instance, all six agents, and the chained workflow
- [ ] Zod schemas for every inter-agent handoff
- [ ] `briefing.worker.ts` invokes the workflow end-to-end, persists `BriefingRun`/`BriefingItem`

### Phase 5 — Scheduling & Delivery
- [ ] `scheduler.ts` (node-cron) enqueues `run-briefing` at 08:00/20:00
- [ ] `admin.routes.ts` manual trigger, admin-gated
- [ ] Per-user personalization filter + `UserBriefing` creation
- [ ] Brevo integration + `DeliveryLog` writes
- [ ] `/briefings`, `/briefings/:id`, `/briefings/:id/read` routes

### Phase 6 — Hardening & Handoff
- [ ] Rate limiting, CORS lockdown, error-shape normalization
- [ ] Load-test a full run against realistic source volume
- [ ] Verify dedup correctness across multiple consecutive runs
- [ ] Freeze OpenAPI spec, write frontend-facing API README
- [ ] Deploy to EC2, confirm PM2 zero-downtime reload works
- [ ] Hand off to frontend development

---

## 21. Startup Intelligence Module (Flagship Feature Spec)

Startup Intelligence is treated as a first-class section of every briefing, not a minor subcategory — reflected directly in the Filter and Summarization agent instructions (§9.2).

**Structured funding story format** (used by the Summarization Agent when `categorySlug = "startups"` and the item is a funding announcement):
```text
Startup: <name>
Funding: <amount> <round type>
Investors: <list>
Valuation: <amount, if disclosed>
Industry: <sector>
Why it matters: <1–2 sentences>
Takeaway: <1 sentence, broader implication>
```

**Additional elements produced by the Trend Analysis Agent for this category:**
- **Funding Trends Snapshot** — aggregate count/total raised per day, broken down by sector.
- **Startup of the Day** — one featured company: what it does, why investors backed it, business model, competitors, funding history.
- **Geographic Distribution** — funding round counts by country/region.
- **Founder Insights** — founder name, prior company/exits, notable quotes, where available.
- **Open Source Startup Watch** — new repos, releases, license changes, CNCF projects tied to startups.

**Recommended high-quality sources:** TechCrunch, Crunchbase, YourStory, Inc42, Entrackr, StartupFeed — consistent, frequent coverage of funding/investor activity.

**Long-term product angle:** because `UserPreference.categorySlugs` already supports arbitrary per-user category selection, this module is the natural seed for eventually positioning the platform as a dedicated startup-intelligence product for a subset of users, without any schema changes.

---

## 22. Newsletter & Web Content Structure

Both the shared Markdown (Newsletter Agent output) and each personalized `UserBriefing` follow this section order — personalization simply removes sections for categories the user doesn't follow:

```text
🔥 Top Headlines
🤖 AI
💻 Technology
🚀 Startup Funding
📊 Funding Trends
⭐ Startup of the Day
🐧 Open Source
📦 GitHub Trending
🔐 Cybersecurity
🌌 Space
🧪 Science
🇮🇳 India
📍 Telangana
🌍 Geopolitics
⚽ Sports
📄 Research Papers
🛠️ Tool of the Day
📅 Upcoming Events
```

Each article entry within a section:
```text
Title
Summary
Why it matters
Source
```

---

## 23. Future Roadmap (V2+)

- **Personalization depth:** per-user topic weighting/ranking (not just on/off categories), learned from `isRead`/engagement over time.
- **Web dashboard features:** search past briefings, bookmark articles, weekly digest view.
- **Notifications:** Slack/Telegram delivery channels alongside email/in-app.
- **Scaling:** move from single-instance BullMQ/Postgres to managed services (e.g. RDS, ElastiCache) if user count or data volume grows meaningfully; introduce read replicas before considering multi-region.
- **Observability:** Prometheus/Grafana once usage justifies the operational overhead.
- **Content quality:** vector-similarity dedup (pgvector) to catch near-duplicate stories worded differently across sources, beyond exact-URL hashing.

---

## 24. Key Architectural Decisions Log

| Decision | Choice | Rationale |
|---|---|---|
| Deployment | AWS EC2, single VPS | Existing familiarity, sufficient for initial scale |
| Containerization | Partial — Docker for stateful services only | Simpler debugging for app processes via PM2 directly on host |
| Repo structure | Separate backend/frontend repos | Clean handoff, independent release cycles |
| API style | REST + OpenAPI | Framework-agnostic contract for any frontend stack |
| Agent framework | Mastra | TypeScript-native; CrewAI has no JS/TS equivalent |
| Auth | Clerk, backend verifies only | No password/session handling burden on this backend |
| Queue | BullMQ + Redis from day one | Multi-user + on-demand triggers require reliable retries immediately, not as a "later" upgrade |
| Dedup strategy | URL-hash at collection, unique constraint at delivery | Two independent, cheap guarantees rather than one complex one |

---

## 25. Glossary

- **Briefing Run** — one execution of the full agent pipeline, producing one shared set of `BriefingItem`s.
- **User Briefing** — the personalized, per-user rendering of a Briefing Run.
- **Agent** — an LLM configured with instructions and optional tools (Mastra primitive).
- **Tool** — a typed, deterministic function an agent can call (Mastra primitive).
- **Workflow** — a typed, multi-step chain of agents/functions (Mastra primitive).
- **Dedup key** — the SHA-256 hash of a normalized article URL, used to prevent duplicate storage/summarization.

---

*This is the complete backend build specification as of the current design discussion. Treat it as a living document — update it whenever an implementation decision changes, so it stays accurate as the single source of truth while building.*