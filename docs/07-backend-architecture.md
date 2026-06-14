# 07 — Backend Architecture

## Overview

Octo Time's backend is a **monolith-first, modular TypeScript/Node.js application** built on Fastify. Every module lives in the same repository and process, but is structured so that any module can be extracted into its own service with minimal refactoring. The design targets a single team operating a single deployment while still meeting the scale requirements of a social media-style tracking platform.

---

## Technology Choices

| Concern | Choice | Rationale |
|---|---|---|
| Runtime | Node.js 22 LTS | Non-blocking I/O fits the heavily read-dominant workload |
| Language | TypeScript 5.x (strict) | Type safety across the entire codebase |
| Framework | Fastify 4 | Lower overhead than Express; schema-first with JSON Schema / Zod integration |
| Primary DB | PostgreSQL 16 | ACID, JSONB for flexible metadata, excellent full-text fallback |
| Cache / Queue | Redis 7 (Valkey-compatible) | Low-latency reads, pub/sub, BullMQ backing |
| Search | Elasticsearch 8 | Multi-field, fuzzy, language-analysed search across anime / TV / movies |
| Message Queue | BullMQ | Redis-backed, TypeScript-native, dead-letter queues, rate limiting |
| Object Storage | Cloudflare R2 (S3-compatible) | No egress fees; Cloudflare CDN co-located |
| CDN | Cloudflare | Media delivery, DDoS mitigation, caching rules |
| Secrets | HashiCorp Vault (or Doppler for small teams) | Environment-specific secrets management |

---

## Architecture Patterns

### Monolith with Module Boundaries

Each functional concern is a **module**. A module owns:
- Its Fastify route plugin (`routes/`)
- Its service classes (`services/`)
- Its repository classes (`repositories/`)
- Its domain types (`types/`)
- Its event emitters / listeners (`events/`)

Modules communicate through a typed **internal event bus** (Node.js `EventEmitter` with a typed wrapper) rather than direct cross-module service calls. This keeps coupling low and makes future extraction straightforward.

### Repository Pattern

All database access is mediated through repository classes. No raw SQL outside repositories. Repositories accept a `DatabaseClient` interface — either a pool connection or a transaction client — enabling transactional composition at the service layer.

```
UserRepository.findById(id, tx?)
UserRepository.updateSettings(id, data, tx?)
```

### Service Layer

Business logic lives in service classes injected with repositories and external API clients. Services orchestrate:
1. Input validation (Zod schemas validated at the route layer)
2. Authorization checks
3. Repository operations
4. Event emission
5. Cache invalidation

### Event-Driven Async

Synchronous HTTP response paths are kept thin. After the response is sent, a typed domain event is emitted which listeners translate into BullMQ jobs for async processing (notifications, stats recomputation, feed updates).

---

## Core Modules

### 1. Auth Module

**Responsibilities:** Registration, login, OAuth social login, session management, token lifecycle.

**JWT Strategy:**
- Access token: 15-minute expiry, signed with RS256 (rotating keypair)
- Refresh token: 30-day expiry, stored as a hash in `user_sessions` table (one row per device)
- Refresh token rotation: each use issues a new refresh token and revokes the old one
- Token family tracking to detect refresh token reuse attacks

**OAuth Providers (Phase 1):** Google, Discord  
**OAuth Providers (Phase 2):** Apple (iOS requirement)

**Routes:**
```
POST /auth/register
POST /auth/login
POST /auth/logout
POST /auth/refresh
POST /auth/oauth/:provider
GET  /auth/oauth/:provider/callback
POST /auth/forgot-password
POST /auth/reset-password
POST /auth/verify-email
```

**Session storage:** Redis hash `session:{userId}:{sessionId}` with expiry = refresh token TTL.

---

### 2. User Module

**Responsibilities:** Public profiles, account settings, follow/unfollow, block list.

**Key tables:** `users`, `user_settings`, `user_follows`, `user_blocks`

**Routes:**
```
GET    /users/:username
PATCH  /users/me
DELETE /users/me
GET    /users/me/settings
PATCH  /users/me/settings
POST   /users/:username/follow
DELETE /users/:username/follow
GET    /users/:username/followers
GET    /users/:username/following
POST   /users/:username/block
DELETE /users/:username/block
```

**Profile cache:** `profile:{username}` in Redis, TTL 5 minutes, invalidated on update.

---

### 3. Media Module

**Responsibilities:** Store and serve normalized anime, TV series, and movie metadata. Scheduled ingestion from AniList (GraphQL) and TMDB (REST).

**Media types:** `anime`, `tv`, `movie`  
**Key tables:** `anime`, `anime_episodes`, `tv_series`, `tv_seasons`, `tv_episodes`, `movies`, `genres`, `studios`, `people`, `characters`, `media_genres`, `media_people`

**Ingestion sources:**
- **AniList:** GraphQL API, paginated media queries, rate-limited (90 req/min)
- **TMDB:** REST API, `/discover`, `/trending`, detail endpoints, rate-limited (40 req/s)

**Ingestion pipeline:**
1. Webhook trigger or scheduled BullMQ job `import_media_data`
2. Fetch raw data from external API
3. Normalize into internal schema via transformer functions
4. Upsert into PostgreSQL (ON CONFLICT UPDATE)
5. Index in Elasticsearch
6. Invalidate Redis cache for affected media

**Routes:**
```
GET /anime
GET /anime/:slug
GET /anime/:slug/episodes
GET /tv
GET /tv/:slug
GET /tv/:slug/seasons/:season/episodes
GET /movies
GET /movies/:slug
GET /characters/:id
GET /people/:id
```

**Media detail cache:** `media:{type}:{slug}` in Redis, TTL 1 hour.

---

### 4. Tracking Module

**Responsibilities:** User's watch list entries, episode/chapter progress, status management.

**Statuses:** `watching`, `completed`, `on_hold`, `dropped`, `plan_to_watch`

**Key tables:** `tracking_entries`, `episode_progress`, `season_progress`

**Data flow — Episode marked as watched:**
```
PATCH /tracking/:mediaType/:mediaId/episodes/:episodeId
  → TrackingService.markEpisodeWatched()
    → validate user owns entry
    → upsert episode_progress row
    → check if season complete → mark season
    → check if series complete → update entry status
    → emit TrackingUpdated event
      → [async] ActivityService.createActivity()
      → [async] NotificationService.notifyFollowers()
      → [async] StatsService.scheduleRecompute()
```

**Routes:**
```
GET    /tracking/me/:mediaType
GET    /tracking/me/:mediaType/:mediaId
POST   /tracking/:mediaType/:mediaId
PATCH  /tracking/:mediaType/:mediaId
DELETE /tracking/:mediaType/:mediaId
PATCH  /tracking/:mediaType/:mediaId/episodes/:episodeId
GET    /tracking/:username/:mediaType  (public, respects privacy)
```

---

### 5. Review Module

**Responsibilities:** Scored reviews, like/reaction system, comments on reviews.

**Key tables:** `reviews`, `review_likes`, `review_comments`, `review_comment_likes`

**Constraints:**
- One review per user per media item (enforced via unique index)
- Reviews can be edited up to 72 hours after posting, then locked
- Minimum 100 characters for a review body
- Score is optional (1–10, 0.5 increments)

**Routes:**
```
POST   /reviews/:mediaType/:mediaId
GET    /reviews/:reviewId
PATCH  /reviews/:reviewId
DELETE /reviews/:reviewId
GET    /reviews/:mediaType/:mediaId  (paginated list)
POST   /reviews/:reviewId/like
DELETE /reviews/:reviewId/like
POST   /reviews/:reviewId/comments
GET    /reviews/:reviewId/comments
PATCH  /reviews/:reviewId/comments/:commentId
DELETE /reviews/:reviewId/comments/:commentId
```

---

### 6. List Module

**Responsibilities:** Custom user lists (e.g. "Top 10 Mecha", "Watch with Friends"), public/private visibility, ordering.

**Key tables:** `lists`, `list_items`, `list_likes`, `list_followers`

**Routes:**
```
POST   /lists
GET    /lists/:slug
PATCH  /lists/:slug
DELETE /lists/:slug
POST   /lists/:slug/items
PATCH  /lists/:slug/items/:itemId
DELETE /lists/:slug/items/:itemId
POST   /lists/:slug/like
DELETE /lists/:slug/like
GET    /users/:username/lists
```

---

### 7. Discovery Module

**Responsibilities:** Trending media, personalized recommendations, seasonal anime calendar, browse by genre/studio.

**Trending computation:**
- BullMQ job `refresh_trending` runs every 15 minutes
- Score formula: `views_24h * 1.0 + list_adds_24h * 2.0 + reviews_24h * 3.0 - age_penalty`
- Results stored in Redis sorted set `trending:{type}` with TTL 15 minutes

**Recommendation approach (Phase 1 — simple):**
- Collaborative filtering based on overlap in completed lists
- Genre and studio affinity from user's watching history
- Computed by `refresh_recommendations` job, stored per user in Redis with TTL 1 hour

**Routes:**
```
GET /discovery/trending/:mediaType
GET /discovery/seasonal          (current anime season)
GET /discovery/recommended       (auth required)
GET /discovery/genres/:genreSlug/:mediaType
GET /discovery/studios/:studioSlug
GET /discovery/new-episodes      (episodes aired this week)
```

---

### 8. Search Module

**Responsibilities:** Full-text search across anime, TV series, movies, users, lists, characters.

**Elasticsearch indices:**
- `media_anime` — title (multilingual), synopsis, genres, studios, year, score
- `media_tv` — title, overview, networks, genres, year, score
- `media_movies` — title, overview, genres, cast, year, score
- `users` — username, display name
- `lists` — title, description (public only)
- `characters` — name, description, media references

**Query strategy:**
- Multi-match across title fields with `best_fields` / `cross_fields`
- Fuzzy matching with `fuzziness: AUTO`
- Boosted exact prefix match on title
- Filter by type, genre, year, status
- Highlight snippets for display

**Cache:** `search:{hash(query+filters)}` in Redis, TTL 5 minutes.

**Sync to Elasticsearch:**
- Real-time on media upsert via event listener
- Nightly full re-index job as safety net

**Routes:**
```
GET /search?q=...&type=...&genre=...&year=...&page=...
GET /search/suggestions?q=...   (autocomplete, lightweight)
```

---

### 9. Sync Module

**Responsibilities:** Import tracking history from AniList, MyAnimeList, Trakt, Letterboxd. Ongoing background sync.

**Supported sources:**
| Source | Media Types | Auth Method |
|---|---|---|
| AniList | Anime, Manga | OAuth2 |
| MyAnimeList | Anime, Manga | OAuth2 (PKCE) |
| Trakt | TV, Movies | OAuth2 |
| Letterboxd | Movies | OAuth2 |

**Sync flow:**
1. User connects external account → OAuth credentials stored encrypted in `user_external_accounts`
2. BullMQ job `sync_user_{source}` queued immediately and then on schedule (daily)
3. Job fetches user's list from external API (paginated)
4. Each item is matched to Octo Time's media by external ID mapping table `external_id_map`
5. Diff computed: new items, status changes, score changes
6. Conflict resolution: most-recent-wins per field (tracked by `synced_at` timestamp)
7. `tracking_entries` updated; if new items, media is imported if not already present
8. Sync result stored in `sync_logs`

**Routes:**
```
GET    /sync/connections              (list connected accounts)
POST   /sync/connect/:source          (initiate OAuth)
DELETE /sync/disconnect/:source
POST   /sync/trigger/:source          (manual sync trigger)
GET    /sync/status/:source           (last sync info)
GET    /sync/logs                     (recent sync history)
```

---

### 10. Notification Module

**Responsibilities:** In-app notifications for follows, reviews, likes, comments, episode releases, sync completion.

**Notification types:** `new_follower`, `review_liked`, `comment_on_review`, `episode_released`, `list_liked`, `sync_complete`, `mention`

**Key tables:** `notifications`, `notification_preferences`

**Delivery:**
- Phase 1: In-app only (WebSocket push to connected clients via Redis pub/sub)
- Phase 2: Push notifications via Expo Notifications (mobile)
- Phase 3: Email digest (via Resend or Postmark)

**Routes:**
```
GET    /notifications
PATCH  /notifications/:id/read
POST   /notifications/read-all
DELETE /notifications/:id
GET    /notifications/preferences
PATCH  /notifications/preferences
```

---

### 11. Statistics Module

**Responsibilities:** Compute and serve per-user stats: total watch time, genre breakdown, completion rate, average score, streak, episodes per month.

**Computation:** Async BullMQ job `compute_user_stats`. Triggered by tracking updates, run fully nightly for all users.

**Output stored in:** `user_stats` (denormalized JSON snapshot), refreshed on demand.

**Stats served:**
- Total minutes watched / episodes watched
- Breakdown by media type
- Top genres, top studios
- Mean score given
- Completion rate per media type
- Watch streak (days with at least one episode marked)
- Monthly watch volume (chart data)

**Routes:**
```
GET /users/:username/stats
GET /users/:username/stats/genres
GET /users/:username/stats/timeline
GET /users/me/stats/recap          (year-in-review style summary)
```

---

### 12. Activity Module

**Responsibilities:** Per-user activity feed, global friend activity feed.

**Activity types:** `started_watching`, `completed`, `rated`, `reviewed`, `added_to_list`, `followed_user`

**Key tables:** `activities`, `activity_privacy` (inherits tracking privacy settings)

**Feed generation:**
- Personal feed: direct DB query on `activities WHERE user_id = ?` ordered by created_at
- Social feed (following): fanout-on-read for MVP (query activities of followed users), fanout-on-write when scale demands it

**Routes:**
```
GET /users/:username/activity
GET /activity/feed              (social feed, auth required)
```

---

### 13. Admin Module

**Responsibilities:** Content moderation, user management, media data corrections, sync management.

**Access control:** `admin` and `moderator` roles on `users.role`

**Capabilities:**
- View/ban/warn users
- Edit media metadata (corrections from community reports)
- Remove reviews / comments violating guidelines
- Trigger manual media imports
- View system health metrics (proxied from Prometheus)
- Manage feature flags

**Routes (all behind `/admin` prefix, role-guarded):**
```
GET    /admin/users
PATCH  /admin/users/:id
POST   /admin/users/:id/ban
POST   /admin/users/:id/warn
GET    /admin/reports
POST   /admin/reports/:id/resolve
PATCH  /admin/media/:type/:id
POST   /admin/sync/trigger
GET    /admin/system/health
```

---

### 14. Export Module

**Responsibilities:** GDPR-compliant data export, migration to/from other services.

**Export formats:** JSON (full), CSV (tracking list), XML (MAL-compatible)

**Flow:**
1. User requests export → `process_export` job queued
2. Job streams all user data (tracking, reviews, lists, activity, settings)
3. Compressed archive (`.zip`) uploaded to R2 with a signed 24-hour URL
4. Notification sent to user with download link

**Routes:**
```
POST /export/request
GET  /export/status/:jobId
GET  /export/download/:jobId   (redirects to signed R2 URL)
```

---

### 15. Webhook Module

**Responsibilities:** Receive push notifications from AniList (new episodes), TMDB (new releases), and internal event forwarding.

**Incoming webhooks:**
- AniList airing schedule → trigger `import_media_data` and notifications for users watching
- TMDB release webhook (via partner program) → update movie/TV data
- Internal health webhooks from Kubernetes liveness probes

**Security:** HMAC-SHA256 signature verification on all incoming webhooks.

**Routes:**
```
POST /webhooks/anilist
POST /webhooks/tmdb
POST /webhooks/internal/:type
```

---

## Infrastructure

### Containerization

**Development (Docker Compose):**
```yaml
services:
  api:          # Fastify app, hot-reload via tsx watch
  postgres:     # PostgreSQL 16
  redis:        # Redis 7
  elasticsearch: # Elasticsearch 8 (single node)
  kibana:       # Dev only
  minio:        # S3-compatible local storage (R2 substitute)
```

**Production (Kubernetes):**
- API: `Deployment` with HPA (min 2, max 20 pods), resource requests/limits set
- PostgreSQL: managed (RDS or Supabase or self-hosted with `CloudNativePG`)
- Redis: managed (Upstash or Redis Cloud) or Kubernetes `StatefulSet`
- Elasticsearch: `Deployment` with persistent volumes (or Elastic Cloud)
- Ingress: nginx-ingress or Cloudflare Tunnel

### CI/CD (GitHub Actions)

```yaml
# .github/workflows/deploy.yml
on:
  push:
    branches: [main]

jobs:
  test:
    - lint (ESLint + Prettier check)
    - type-check (tsc --noEmit)
    - unit tests (Vitest)
    - integration tests (Postgres + Redis test containers)

  build:
    - docker build --target production
    - push to GHCR (GitHub Container Registry)

  deploy:
    - kubectl set image deployment/api api=ghcr.io/octotime/api:$SHA
    - kubectl rollout status deployment/api
```

### Monitoring

| Tool | Purpose |
|---|---|
| Prometheus | Metrics scraping (Fastify metrics plugin exposes `/metrics`) |
| Grafana | Dashboards: RPS, P99 latency, DB pool utilization, queue depth |
| Loki | Log aggregation (structured JSON logs shipped via Promtail) |
| Sentry | Error tracking and performance tracing (Sentry SDK for Node.js) |
| Uptime Kuma | External uptime checks |

**Key Grafana dashboards:**
- API overview (RPS, latency P50/P95/P99, error rate)
- Database pool (active connections, query time, slow queries)
- Queue health (job throughput, failures, lag per queue)
- Cache hit rate (Redis HIT/MISS ratio per key pattern)
- Sync jobs (success/failure rate per external source)

### Health Checks

```typescript
// Fastify health plugin
GET /health/live    → always 200 (liveness probe)
GET /health/ready   → checks: postgres connection, redis ping, ES ping
GET /health/startup → checks: DB migrations applied, required config present
```

Kubernetes probe configuration:
```yaml
livenessProbe:
  httpGet: { path: /health/live, port: 3000 }
  initialDelaySeconds: 10
  periodSeconds: 15

readinessProbe:
  httpGet: { path: /health/ready, port: 3000 }
  initialDelaySeconds: 5
  periodSeconds: 10
```

---

## Caching Strategy

| Cache Key Pattern | Store | TTL | Invalidation Trigger |
|---|---|---|---|
| `profile:{username}` | Redis | 5 min | User settings update |
| `media:{type}:{slug}` | Redis | 1 hr | Media ingestion update |
| `trending:{type}` | Redis | 15 min | `refresh_trending` job |
| `session:{uid}:{sid}` | Redis | 30 days | Logout or rotation |
| `search:{queryHash}` | Redis | 5 min | TTL only |
| `recommendations:{uid}` | Redis | 1 hr | `refresh_recommendations` job |
| `stats:{uid}` | Redis | 10 min | `compute_user_stats` job |
| `feed:{uid}:{page}` | Redis | 2 min | New activity by followed user |

**Cache population strategy:** Cache-aside (lazy loading). Data is fetched from DB and written to cache on the first miss.

**Cache warming:** Trending and seasonal data is proactively populated by scheduled jobs before TTL expires to avoid thundering herd.

---

## Queue Jobs (BullMQ)

### Queue Definitions

```typescript
// Queues
const syncQueue       = new Queue('sync',        { connection: redis })
const statsQueue      = new Queue('stats',       { connection: redis })
const notifyQueue     = new Queue('notify',      { connection: redis })
const mediaQueue      = new Queue('media',       { connection: redis })
const exportQueue     = new Queue('export',      { connection: redis })
const discoveryQueue  = new Queue('discovery',   { connection: redis })
```

### Job Definitions

| Job Name | Queue | Trigger | Retry Policy |
|---|---|---|---|
| `sync_user_anilist` | sync | User connects / daily schedule | 3 retries, exp backoff |
| `sync_user_mal` | sync | User connects / daily schedule | 3 retries, exp backoff |
| `sync_user_trakt` | sync | User connects / daily schedule | 3 retries, exp backoff |
| `sync_user_letterboxd` | sync | User connects / daily schedule | 3 retries, exp backoff |
| `compute_user_stats` | stats | Tracking update event | 2 retries, 5s delay |
| `send_notification` | notify | Domain events | 3 retries, 1s delay |
| `process_export` | export | User export request | 1 retry, alert on failure |
| `import_media_data` | media | Webhook / manual trigger | 3 retries |
| `refresh_trending` | discovery | Cron: every 15 min | 1 retry |
| `refresh_recommendations` | discovery | Cron: every 1 hr | 1 retry |

### Dead Letter Queue

Failed jobs after max retries move to `{queue}-failed`. A Grafana alert fires when the DLQ length exceeds 50. Failed jobs are inspectable via Bull Board (`/admin/queues`, admin-only).

---

## Security

### Rate Limiting

```typescript
// Per-IP (unauthenticated)
fastify.register(rateLimitPlugin, {
  max: 100,
  timeWindow: '1 minute',
  keyGenerator: (req) => req.ip,
})

// Per-user (authenticated, higher limits)
// Applied per route group:
// - Auth routes: 10/min (anti-brute-force)
// - Search: 60/min
// - Write routes: 30/min
// - Read routes: 300/min
```

### Input Validation

All incoming request bodies and query params are validated with Zod schemas. Fastify's `setValidatorCompiler` is replaced with a Zod adapter to keep schema definitions in TypeScript.

```typescript
const CreateReviewSchema = z.object({
  body: z.string().min(100).max(10000),
  score: z.number().min(1).max(10).multipleOf(0.5).optional(),
  containsSpoilers: z.boolean().default(false),
})
```

### Security Headers (Helmet.js)

```
Content-Security-Policy: default-src 'self'; img-src 'self' data: cdn.octotime.app;
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=()
```

### SQL Injection

All queries use parameterized statements via the `pg` driver or the Drizzle ORM query builder. Raw SQL is forbidden in application code. The `no-raw-sql` ESLint rule enforces this.

### CORS

```typescript
fastify.register(cors, {
  origin: [
    'https://octotime.app',
    'https://www.octotime.app',
    process.env.NODE_ENV === 'development' ? 'http://localhost:3001' : false,
  ].filter(Boolean),
  credentials: true,
  methods: ['GET', 'POST', 'PATCH', 'DELETE', 'OPTIONS'],
})
```

---

## Data Flow Examples

### Episode Marked as Watched

```
Client → PATCH /tracking/anime/123/episodes/456
  → AuthGuard (validate JWT, attach user)
  → ZodValidation
  → TrackingController.markEpisodeWatched()
    → TrackingService.markEpisodeWatched(userId, animeId, episodeId)
      → TrackingRepository.upsertEpisodeProgress(tx)
      → TrackingRepository.checkSeasonCompletion(tx)
      → if complete: TrackingRepository.markSeasonComplete(tx)
      → TrackingRepository.checkSeriesCompletion(tx)
      → if complete: TrackingRepository.updateEntryStatus('completed', tx)
      → commit transaction
      → emit TrackingEpisodeWatched { userId, animeId, episodeId, timestamp }
        → [BullMQ] ActivityWorker.createActivity()
        → [BullMQ] NotificationWorker.notifyFollowers()
        → [BullMQ] StatsWorker.scheduleRecompute()
  → Response 200 { progress }
```

### External Sync

```
[BullMQ Worker] sync_user_anilist job received
  → AniListClient.fetchUserList(accessToken, page)   (paginated)
  → AniListTransformer.normalize(rawList)             → ExternalEntry[]
  → ExternalIdMap.resolveToLocalIds(externalIds)
  → for each entry:
      → diff against existing TrackingEntry
      → if changed: TrackingRepository.update() with conflict resolution
      → if new: TrackingRepository.create()
      → if media not found: queue import_media_data job
  → SyncLogRepository.recordSuccess(source, userId, count)
  → emit SyncCompleted event → notification to user
```

### Search Query

```
Client → GET /search?q=attack+on+titan&type=anime
  → SearchController.search()
    → Cache.get('search:{hash}') → HIT → return cached
    → MISS →
      → ElasticsearchService.search({
          indices: ['media_anime'],
          query: { multi_match: { query: 'attack on titan', ... } },
          highlight: { fields: { title: {} } },
          size: 20, from: 0
        })
      → SearchResultTransformer.toResponse(esHits)
      → Cache.set('search:{hash}', result, ttl=300)
    → Response 200 { results, total, took }
```

---

## Scalability Design

### Horizontal API Scaling

API pods are stateless (all state in PostgreSQL/Redis). HPA scales on CPU (target 70%) and custom metric `fastify_request_queue_depth`. Session and rate limit state lives in Redis, shared across all pods.

### PostgreSQL Read Replicas

Read-heavy queries (media detail, profile pages, search fallback) are routed to one of N read replicas via the `pg-pool-read` connection pool. The repository layer accepts a `useReplica: boolean` option. Writes always go to the primary.

### Redis Cluster

Redis is deployed in cluster mode with 3 primaries + 3 replicas. BullMQ is compatible with Redis cluster. Cache key sharding is handled transparently.

### Elasticsearch Cluster

Minimum 3-node cluster (1 master-eligible + 2 data nodes for MVP). Index settings:
- `media_anime`: 2 primary shards, 1 replica
- `media_tv`, `media_movies`: 1 primary shard, 1 replica
- `users`, `lists`, `characters`: 1 primary shard, 1 replica

---

## Folder Structure

```
apps/api/
├── src/
│   ├── main.ts                    # App entry point
│   ├── app.ts                     # Fastify instance, plugin registration
│   ├── config/
│   │   ├── env.ts                 # Zod-validated environment config
│   │   ├── database.ts            # PostgreSQL pool setup
│   │   ├── redis.ts               # Redis client setup
│   │   ├── elasticsearch.ts       # ES client setup
│   │   └── storage.ts             # R2 / S3 client setup
│   ├── plugins/
│   │   ├── auth.ts                # JWT plugin, auth decorator
│   │   ├── cors.ts
│   │   ├── rate-limit.ts
│   │   ├── helmet.ts
│   │   ├── sentry.ts
│   │   └── metrics.ts             # Prometheus metrics
│   ├── modules/
│   │   ├── auth/
│   │   │   ├── auth.routes.ts
│   │   │   ├── auth.service.ts
│   │   │   ├── auth.repository.ts
│   │   │   ├── auth.types.ts
│   │   │   └── auth.schemas.ts    # Zod schemas
│   │   ├── user/
│   │   ├── media/
│   │   │   ├── media.routes.ts
│   │   │   ├── anime/
│   │   │   │   ├── anime.service.ts
│   │   │   │   ├── anime.repository.ts
│   │   │   │   └── anilist.client.ts
│   │   │   ├── tv/
│   │   │   │   ├── tv.service.ts
│   │   │   │   ├── tv.repository.ts
│   │   │   │   └── tmdb.client.ts
│   │   │   └── movies/
│   │   ├── tracking/
│   │   ├── review/
│   │   ├── list/
│   │   ├── discovery/
│   │   ├── search/
│   │   ├── sync/
│   │   ├── notification/
│   │   ├── stats/
│   │   ├── activity/
│   │   ├── admin/
│   │   ├── export/
│   │   └── webhook/
│   ├── workers/
│   │   ├── sync.worker.ts
│   │   ├── stats.worker.ts
│   │   ├── notify.worker.ts
│   │   ├── media.worker.ts
│   │   ├── export.worker.ts
│   │   └── discovery.worker.ts
│   ├── events/
│   │   ├── event-bus.ts           # Typed EventEmitter wrapper
│   │   └── event-types.ts         # Domain event type definitions
│   ├── db/
│   │   ├── schema/                # Drizzle ORM schema files
│   │   │   ├── users.ts
│   │   │   ├── media.ts
│   │   │   ├── tracking.ts
│   │   │   └── ...
│   │   ├── migrations/            # SQL migration files
│   │   └── seed/                  # Dev seed scripts
│   ├── shared/
│   │   ├── errors.ts              # Typed HTTP error classes
│   │   ├── pagination.ts          # Cursor/offset pagination helpers
│   │   ├── cache.ts               # Cache helper (get/set/invalidate)
│   │   └── logger.ts              # Pino logger instance
│   └── types/
│       └── fastify.d.ts           # Module augmentation for request decorators
├── tests/
│   ├── unit/
│   ├── integration/               # Uses testcontainers for PG + Redis
│   └── fixtures/
├── Dockerfile
├── docker-compose.yml
├── drizzle.config.ts
├── vitest.config.ts
├── tsconfig.json
└── package.json
```

---

## Environment Configuration

```bash
# Server
NODE_ENV=production
PORT=3000
HOST=0.0.0.0
API_BASE_URL=https://api.octotime.app

# Database
DATABASE_URL=postgresql://user:pass@host:5432/octotime
DATABASE_REPLICA_URL=postgresql://user:pass@replica:5432/octotime
DATABASE_POOL_MIN=5
DATABASE_POOL_MAX=20

# Redis
REDIS_URL=redis://host:6379
REDIS_CLUSTER_NODES=host1:6379,host2:6379,host3:6379

# Elasticsearch
ELASTICSEARCH_URL=https://host:9200
ELASTICSEARCH_API_KEY=...

# JWT
JWT_ACCESS_SECRET=...       # RS256 private key (PEM)
JWT_REFRESH_SECRET=...      # HS256 shared secret (256-bit random)
JWT_ACCESS_TTL=900          # 15 minutes in seconds
JWT_REFRESH_TTL=2592000     # 30 days in seconds

# OAuth
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...
DISCORD_CLIENT_ID=...
DISCORD_CLIENT_SECRET=...
APPLE_CLIENT_ID=...
APPLE_PRIVATE_KEY=...

# External APIs
ANILIST_CLIENT_ID=...
ANILIST_CLIENT_SECRET=...
TMDB_API_KEY=...
MAL_CLIENT_ID=...
MAL_CLIENT_SECRET=...
TRAKT_CLIENT_ID=...
TRAKT_CLIENT_SECRET=...
LETTERBOXD_CLIENT_ID=...
LETTERBOXD_CLIENT_SECRET=...

# Storage
R2_ACCOUNT_ID=...
R2_ACCESS_KEY_ID=...
R2_SECRET_ACCESS_KEY=...
R2_BUCKET_NAME=octotime-media
R2_PUBLIC_URL=https://media.octotime.app

# Monitoring
SENTRY_DSN=...
PROMETHEUS_PORT=9090

# Email (Phase 2)
RESEND_API_KEY=...
EMAIL_FROM=noreply@octotime.app
```

---

## Startup Sequence

```typescript
// main.ts
async function bootstrap() {
  // 1. Validate environment (Zod — throws on missing required vars)
  const config = parseEnv(process.env)

  // 2. Connect to infrastructure
  await database.connect()
  await redis.ping()
  await elasticsearch.ping()

  // 3. Run pending database migrations
  await runMigrations(database)

  // 4. Create Fastify instance
  const app = buildApp(config)

  // 5. Register plugins (order matters)
  await app.register(helmetPlugin)
  await app.register(corsPlugin)
  await app.register(rateLimitPlugin)
  await app.register(metricsPlugin)
  await app.register(authPlugin)
  await app.register(sentryPlugin)

  // 6. Register module routes
  await app.register(authRoutes,         { prefix: '/auth' })
  await app.register(userRoutes,         { prefix: '/users' })
  await app.register(mediaRoutes,        { prefix: '/media' })
  await app.register(trackingRoutes,     { prefix: '/tracking' })
  await app.register(reviewRoutes,       { prefix: '/reviews' })
  await app.register(listRoutes,         { prefix: '/lists' })
  await app.register(searchRoutes,       { prefix: '/search' })
  await app.register(discoveryRoutes,    { prefix: '/discovery' })
  await app.register(syncRoutes,         { prefix: '/sync' })
  await app.register(notificationRoutes, { prefix: '/notifications' })
  await app.register(statsRoutes,        { prefix: '/stats' })
  await app.register(activityRoutes,     { prefix: '/activity' })
  await app.register(exportRoutes,       { prefix: '/export' })
  await app.register(webhookRoutes,      { prefix: '/webhooks' })
  await app.register(adminRoutes,        { prefix: '/admin' })
  await app.register(healthRoutes,       { prefix: '/health' })

  // 7. Start BullMQ workers
  await startWorkers(config)

  // 8. Schedule recurring jobs
  await scheduleRecurringJobs()

  // 9. Start listening
  await app.listen({ port: config.PORT, host: config.HOST })
  app.log.info(`API running on port ${config.PORT}`)

  // 10. Graceful shutdown handlers
  process.on('SIGTERM', () => gracefulShutdown(app))
  process.on('SIGINT',  () => gracefulShutdown(app))
}

bootstrap().catch((err) => {
  console.error('Failed to start:', err)
  process.exit(1)
})
```
