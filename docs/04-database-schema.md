# Octo Time — Database Schema

## Table of Contents

1. [Technology Stack & Rationale](#1-technology-stack--rationale)
2. [Naming Conventions](#2-naming-conventions)
3. [Enum Type Definitions](#3-enum-type-definitions)
4. [Core User Tables](#4-core-user-tables)
5. [Social Auth Providers](#5-social-auth-providers)
6. [Social Graph](#6-social-graph)
7. [Media Tables](#7-media-tables)
8. [Seasons & Episodes](#8-seasons--episodes)
9. [People, Characters & Cast](#9-people-characters--cast)
10. [Studios & Networks](#10-studios--networks)
11. [Genres](#11-genres)
12. [User Media Tracking](#12-user-media-tracking)
13. [Reviews, Comments & Likes](#13-reviews-comments--likes)
14. [Lists & Favorites](#14-lists--favorites)
15. [Notifications & Activity Feed](#15-notifications--activity-feed)
16. [Import / Sync System](#16-import--sync-system)
17. [Statistics & Cached Aggregates](#17-statistics--cached-aggregates)
18. [Media Screenshots](#18-media-screenshots)
19. [Admin & Moderation](#19-admin--moderation)
20. [Full-Text Search Configuration](#20-full-text-search-configuration)
21. [Row-Level Security Policies](#21-row-level-security-policies)
22. [Trigger Definitions](#22-trigger-definitions)
23. [Partitioning Strategy](#23-partitioning-strategy)
24. [Index Strategy Summary](#24-index-strategy-summary)
25. [Migration Strategy](#25-migration-strategy)
26. [Data Retention Policies](#26-data-retention-policies)
27. [Soft Delete Pattern](#27-soft-delete-pattern)

---

## 1. Technology Stack & Rationale

### Primary Database: PostgreSQL 16

PostgreSQL is chosen as the primary transactional store for the following reasons:

- **ACID compliance** — media tracking inherently involves multi-table writes (e.g., marking an episode watched must atomically update `user_episode_progress`, `user_media_entry`, and `user_stats`). PostgreSQL's serializable isolation prevents phantom reads and write skews that would corrupt watch counts.
- **JSONB support** — notification payloads, sync conflict data, and activity metadata are semi-structured. JSONB provides schema-flexible storage with GIN indexing for fast key-value lookups, eliminating the need for a separate document store for these cases.
- **Rich type system** — native `ENUM`, `DECIMAL`, `ARRAY`, `TIMESTAMPTZ`, and `UUID` types map directly to domain concepts without application-layer translation overhead.
- **Partitioning** — the `activities` and `notifications` tables are partitioned by month using declarative range partitioning, enabling cheap partition pruning for recent-data queries and zero-downtime old-data archival.
- **Full-text search** — `tsvector` columns and GIN indexes power cross-language title search (English + Arabic) for anime, TV series, and movies without requiring Elasticsearch for basic searches.
- **Row-Level Security** — privacy settings (`public`/`private`) are enforced in the database layer, making it impossible for application bugs to leak private user data.
- **Mature ecosystem** — pg_partman for partition management, pg_cron for scheduled jobs, pgcrypto for UUID generation, and PostGIS (future roadmap for location-based features) are all native extensions.

### Cache Layer: Redis 7

Redis is used for:

- **Session storage** — JWT refresh token blacklisting and OAuth state parameters (TTL: 10 minutes for OAuth state, 30 days for refresh token records).
- **Rate limiting** — sliding window counters per `(user_id, endpoint)` to prevent abuse of write-heavy operations (reviews, follows, list edits).
- **Hot media cache** — trending anime/TV/movie lists recalculated every 15 minutes and served from Redis to avoid repeated aggregation queries.
- **UserStats read cache** — the `user_stats` table is the source of truth, but Redis mirrors the values with a 5-minute TTL to serve profile stats without a database hit.
- **Notification unread counts** — incremented on insert trigger notification, decremented on read, stored as a Redis integer per `user_id`.
- **Pub/Sub** — real-time activity fan-out to followers' feeds uses Redis Streams. Consumers (Go workers) write to the `activities` table in the background.

### Search: Elasticsearch 8

Elasticsearch supplements PostgreSQL's built-in full-text search for:

- **Fuzzy title search** — "Naurto" should still find Naruto. PostgreSQL `similarity()` (pg_trgm) handles this at small scale, but Elasticsearch's BM25 + fuzziness provides better relevance ranking at scale.
- **Multi-field cross-entity search** — a single search box returning ranked results across anime, TV series, movies, characters, and people requires a unified index that spans entity types.
- **Autocomplete / type-ahead** — edge-ngram tokenization provides sub-50ms prefix search for the search bar.
- **Faceted search** — filtering by genre, year, studio, format simultaneously with scored relevance ranking is not ergonomic in PostgreSQL.

The canonical source of truth for all data remains PostgreSQL. Elasticsearch is populated by a change-data-capture pipeline (Debezium + Kafka) that tails the PostgreSQL WAL and syncs inserts/updates/deletes asynchronously with eventual consistency (typically < 1 second lag).

---

## 2. Naming Conventions

- Table names: `snake_case`, plural (e.g., `users`, `anime_episodes`).
- Column names: `snake_case`.
- Primary keys: always `id UUID DEFAULT gen_random_uuid()`.
- Foreign keys: `<referenced_table_singular>_id` (e.g., `user_id`, `season_id`).
- Enum types: `<domain>_<attribute>_enum` (e.g., `media_type_enum`, `sync_provider_enum`).
- Indexes: `idx_<table>_<columns>` for plain B-tree, `idx_<table>_<column>_gin` for GIN, `idx_<table>_<column>_fts` for full-text.
- Constraints: `chk_<table>_<rule>`, `uq_<table>_<columns>`, `fk_<table>_<column>`.

---

## 3. Enum Type Definitions

```sql
-- ============================================================
-- ENUM TYPES
-- All enums are defined before any table that references them.
-- ============================================================

CREATE TYPE media_type_enum AS ENUM (
    'anime',
    'tv_series',
    'movie'
);

CREATE TYPE anime_status_enum AS ENUM (
    'airing',
    'finished',
    'not_yet_aired',
    'cancelled',
    'hiatus'
);

CREATE TYPE anime_format_enum AS ENUM (
    'tv',
    'movie',
    'ova',
    'ona',
    'special',
    'music',
    'manga',
    'novel',
    'one_shot'
);

CREATE TYPE anime_source_enum AS ENUM (
    'manga',
    'light_novel',
    'visual_novel',
    'video_game',
    'original',
    'other',
    'novel',
    'doujinshi',
    'anime',
    'web_novel',
    'live_action',
    'card_game',
    'book',
    'music'
);

CREATE TYPE tv_series_status_enum AS ENUM (
    'returning_series',
    'ended',
    'cancelled',
    'in_production',
    'planned',
    'pilot',
    'airing'
);

CREATE TYPE movie_status_enum AS ENUM (
    'released',
    'in_production',
    'post_production',
    'planned',
    'cancelled',
    'rumored'
);

CREATE TYPE user_media_status_enum AS ENUM (
    'watching',
    'completed',
    'planned',
    'paused',
    'dropped',
    'rewatching'
);

CREATE TYPE privacy_setting_enum AS ENUM (
    'public',
    'private',
    'followers_only'
);

CREATE TYPE list_visibility_enum AS ENUM (
    'public',
    'private',
    'followers_only'
);

CREATE TYPE notification_type_enum AS ENUM (
    'review_like',
    'comment_reply',
    'new_follower',
    'new_episode',
    'new_season',
    'new_comment_on_review',
    'mention'
);

CREATE TYPE activity_type_enum AS ENUM (
    'watched_episode',
    'completed_movie',
    'completed_series',
    'completed_anime',
    'rating',
    'review',
    'new_list',
    'added_to_list',
    'started_watching',
    'dropped',
    'rewatching'
);

CREATE TYPE sync_provider_enum AS ENUM (
    'anilist',
    'mal',
    'trakt',
    'letterboxd'
);

CREATE TYPE sync_job_status_enum AS ENUM (
    'pending',
    'running',
    'completed',
    'failed',
    'partial'
);

CREATE TYPE sync_resolution_enum AS ENUM (
    'keep_local',
    'keep_remote',
    'merged',
    'skipped'
);

CREATE TYPE oauth_provider_enum AS ENUM (
    'google',
    'anilist',
    'mal',
    'trakt',
    'letterboxd'
);

CREATE TYPE admin_action_type_enum AS ENUM (
    'remove_review',
    'remove_comment',
    'ban_user',
    'unban_user',
    'edit_media',
    'delete_list',
    'warn_user'
);

CREATE TYPE theme_enum AS ENUM (
    'light',
    'dark',
    'system'
);

CREATE TYPE locale_enum AS ENUM (
    'en',
    'ar'
);

CREATE TYPE screenshot_type_enum AS ENUM (
    'screenshot',
    'poster',
    'backdrop',
    'logo',
    'banner'
);

CREATE TYPE season_enum AS ENUM (
    'winter',
    'spring',
    'summer',
    'fall'
);

CREATE TYPE favorites_media_type_enum AS ENUM (
    'tv_series',
    'movie'
);
```

---

## 4. Core User Tables

```sql
-- ============================================================
-- EXTENSION SETUP
-- ============================================================

CREATE EXTENSION IF NOT EXISTS "pgcrypto";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";
CREATE EXTENSION IF NOT EXISTS "unaccent";
CREATE EXTENSION IF NOT EXISTS "btree_gin";

-- ============================================================
-- TABLE: users
-- Central identity record. Soft-deletable via deleted_at.
-- ============================================================

CREATE TABLE users (
    id                  UUID            NOT NULL DEFAULT gen_random_uuid(),
    username            VARCHAR(30)     NOT NULL,
    display_name        VARCHAR(60)     NOT NULL,
    email               VARCHAR(255)    NOT NULL,
    password_hash       VARCHAR(255)    NULL,       -- NULL when user signed up via OAuth only
    avatar_url          TEXT            NULL,
    banner_url          TEXT            NULL,
    locale              locale_enum     NOT NULL DEFAULT 'en',
    theme               theme_enum      NOT NULL DEFAULT 'system',
    privacy_setting     privacy_setting_enum NOT NULL DEFAULT 'public',
    is_verified         BOOLEAN         NOT NULL DEFAULT FALSE,
    is_admin            BOOLEAN         NOT NULL DEFAULT FALSE,
    is_banned           BOOLEAN         NOT NULL DEFAULT FALSE,
    ban_reason          TEXT            NULL,
    email_verified_at   TIMESTAMPTZ     NULL,
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    deleted_at          TIMESTAMPTZ     NULL,

    CONSTRAINT pk_users PRIMARY KEY (id),
    CONSTRAINT uq_users_username UNIQUE (username),
    CONSTRAINT uq_users_email UNIQUE (email),
    CONSTRAINT chk_users_username_length CHECK (char_length(username) >= 3),
    CONSTRAINT chk_users_username_chars CHECK (username ~ '^[a-zA-Z0-9_]+$'),
    CONSTRAINT chk_users_display_name_length CHECK (char_length(display_name) >= 1),
    CONSTRAINT chk_users_email_format CHECK (email ~ '^[^@\s]+@[^@\s]+\.[^@\s]+$'),
    CONSTRAINT chk_users_deleted_consistency CHECK (
        deleted_at IS NULL OR (deleted_at IS NOT NULL AND is_banned = FALSE)
    )
);

-- Indexes on users
CREATE INDEX idx_users_username ON users (username) WHERE deleted_at IS NULL;
CREATE INDEX idx_users_email ON users (email) WHERE deleted_at IS NULL;
CREATE INDEX idx_users_created_at ON users (created_at DESC);
CREATE INDEX idx_users_deleted_at ON users (deleted_at) WHERE deleted_at IS NOT NULL;
CREATE INDEX idx_users_is_admin ON users (is_admin) WHERE is_admin = TRUE;

-- Auto-update updated_at
CREATE OR REPLACE FUNCTION fn_set_updated_at()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_users_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION fn_set_updated_at();

-- ============================================================
-- TABLE: user_preferences
-- One-to-one with users. Stores notification toggles,
-- display preferences, and import/export settings.
-- ============================================================

CREATE TABLE user_preferences (
    user_id                         UUID        NOT NULL,
    notify_review_like              BOOLEAN     NOT NULL DEFAULT TRUE,
    notify_comment_reply            BOOLEAN     NOT NULL DEFAULT TRUE,
    notify_new_follower             BOOLEAN     NOT NULL DEFAULT TRUE,
    notify_new_episode              BOOLEAN     NOT NULL DEFAULT TRUE,
    notify_new_season               BOOLEAN     NOT NULL DEFAULT TRUE,
    show_adult_content              BOOLEAN     NOT NULL DEFAULT FALSE,
    default_list_visibility         list_visibility_enum NOT NULL DEFAULT 'public',
    show_activity_feed              BOOLEAN     NOT NULL DEFAULT TRUE,
    show_watch_stats_publicly       BOOLEAN     NOT NULL DEFAULT TRUE,
    show_reviews_publicly           BOOLEAN     NOT NULL DEFAULT TRUE,
    show_lists_publicly             BOOLEAN     NOT NULL DEFAULT TRUE,
    language_filter                 TEXT[]      NOT NULL DEFAULT '{}',   -- preferred audio/sub languages
    autoplay_trailers               BOOLEAN     NOT NULL DEFAULT TRUE,
    created_at                      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at                      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT pk_user_preferences PRIMARY KEY (user_id),
    CONSTRAINT fk_user_preferences_user FOREIGN KEY (user_id)
        REFERENCES users (id) ON DELETE CASCADE
);

CREATE TRIGGER trg_user_preferences_updated_at
    BEFORE UPDATE ON user_preferences
    FOR EACH ROW EXECUTE FUNCTION fn_set_updated_at();

-- Auto-create preferences on user insert
CREATE OR REPLACE FUNCTION fn_create_user_preferences()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    INSERT INTO user_preferences (user_id) VALUES (NEW.id);
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_users_create_preferences
    AFTER INSERT ON users
    FOR EACH ROW EXECUTE FUNCTION fn_create_user_preferences();
```

---

## 5. Social Auth Providers

```sql
-- ============================================================
-- TABLE: user_oauth_connections
-- One user can have multiple OAuth connections.
-- Each (user_id, provider) pair must be unique.
-- The provider_user_id is the external platform's user ID
-- (e.g., AniList's viewer.id, Google's sub claim).
-- ============================================================

CREATE TABLE user_oauth_connections (
    id                  UUID            NOT NULL DEFAULT gen_random_uuid(),
    user_id             UUID            NOT NULL,
    provider            oauth_provider_enum NOT NULL,
    provider_user_id    VARCHAR(255)    NOT NULL,
    provider_username   VARCHAR(255)    NULL,       -- display name on the external platform
    access_token        TEXT            NULL,       -- encrypted at rest via pgcrypto
    refresh_token       TEXT            NULL,       -- encrypted at rest via pgcrypto
    token_expires_at    TIMESTAMPTZ     NULL,
    scopes              TEXT[]          NOT NULL DEFAULT '{}',
    connected_at        TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),

    CONSTRAINT pk_user_oauth_connections PRIMARY KEY (id),
    CONSTRAINT uq_user_oauth_provider UNIQUE (user_id, provider),
    CONSTRAINT uq_oauth_provider_external UNIQUE (provider, provider_user_id),
    CONSTRAINT fk_user_oauth_connections_user FOREIGN KEY (user_id)
        REFERENCES users (id) ON DELETE CASCADE
);

CREATE INDEX idx_user_oauth_connections_user_id ON user_oauth_connections (user_id);
CREATE INDEX idx_user_oauth_connections_provider ON user_oauth_connections (provider, provider_user_id);

CREATE TRIGGER trg_user_oauth_connections_updated_at
    BEFORE UPDATE ON user_oauth_connections
    FOR EACH ROW EXECUTE FUNCTION fn_set_updated_at();
```

---

## 6. Social Graph

```sql
-- ============================================================
-- TABLE: follows
-- Directed graph edge: follower_id follows following_id.
-- Self-follow is prevented by check constraint.
-- ============================================================

CREATE TABLE follows (
    follower_id         UUID            NOT NULL,
    following_id        UUID            NOT NULL,
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),

    CONSTRAINT pk_follows PRIMARY KEY (follower_id, following_id),
    CONSTRAINT fk_follows_follower FOREIGN KEY (follower_id)
        REFERENCES users (id) ON DELETE CASCADE,
    CONSTRAINT fk_follows_following FOREIGN KEY (following_id)
        REFERENCES users (id) ON DELETE CASCADE,
    CONSTRAINT chk_follows_no_self_follow CHECK (follower_id <> following_id)
);

-- Lookup "who does user X follow?"
CREATE INDEX idx_follows_follower_id ON follows (follower_id);
-- Lookup "who follows user X?"
CREATE INDEX idx_follows_following_id ON follows (following_id);
-- Ordered by join date for paginated follower lists
CREATE INDEX idx_follows_following_created ON follows (following_id, created_at DESC);
CREATE INDEX idx_follows_follower_created ON follows (follower_id, created_at DESC);

-- Materialized follower/following counts cached in user_stats (see §17)
```

---

## 7. Media Tables

```sql
-- ============================================================
-- TABLE: anime
-- Sourced from AniList API. anilist_id is the stable external
-- identifier used for import/sync operations.
-- Genres and studios are normalized into separate tables and
-- linked via junction tables for filtering efficiency.
-- ============================================================

CREATE TABLE anime (
    id                  UUID            NOT NULL DEFAULT gen_random_uuid(),
    anilist_id          INTEGER         NOT NULL,
    title_en            TEXT            NULL,
    title_ar            TEXT            NULL,
    title_native        TEXT            NULL,       -- Japanese title in native script
    title_romaji        TEXT            NULL,       -- Romanized Japanese title
    synopsis_en         TEXT            NULL,
    synopsis_ar         TEXT            NULL,
    cover_image         TEXT            NULL,       -- Full URL to cover image
    banner_image        TEXT            NULL,       -- Full URL to banner image
    trailer_url         TEXT            NULL,       -- YouTube or other video URL
    status              anime_status_enum NULL,
    format              anime_format_enum NULL,
    episode_count       SMALLINT        NULL CHECK (episode_count > 0),
    episode_duration    SMALLINT        NULL CHECK (episode_duration > 0),  -- minutes
    start_date          DATE            NULL,
    end_date            DATE            NULL,
    season              season_enum     NULL,
    season_year         SMALLINT        NULL CHECK (season_year BETWEEN 1900 AND 2100),
    is_adult            BOOLEAN         NOT NULL DEFAULT FALSE,
    average_rating      DECIMAL(4,2)    NULL CHECK (average_rating BETWEEN 0 AND 100),  -- AniList uses 0-100
    popularity          INTEGER         NOT NULL DEFAULT 0 CHECK (popularity >= 0),
    source              anime_source_enum NULL,
    country_of_origin   CHAR(2)         NULL,       -- ISO 3166-1 alpha-2
    hashtag             VARCHAR(100)    NULL,
    -- Full-text search vectors (updated by trigger)
    search_vector       TSVECTOR        GENERATED ALWAYS AS (
        setweight(to_tsvector('english', coalesce(title_en, '')), 'A') ||
        setweight(to_tsvector('simple', coalesce(title_romaji, '')), 'A') ||
        setweight(to_tsvector('simple', coalesce(title_native, '')), 'B') ||
        setweight(to_tsvector('english', coalesce(synopsis_en, '')), 'C')
    ) STORED,
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),

    CONSTRAINT pk_anime PRIMARY KEY (id),
    CONSTRAINT uq_anime_anilist_id UNIQUE (anilist_id),
    CONSTRAINT chk_anime_dates CHECK (end_date IS NULL OR start_date IS NULL OR end_date >= start_date)
);

CREATE INDEX idx_anime_anilist_id ON anime (anilist_id);
CREATE INDEX idx_anime_status ON anime (status);
CREATE INDEX idx_anime_format ON anime (format);
CREATE INDEX idx_anime_season ON anime (season, season_year);
CREATE INDEX idx_anime_popularity ON anime (popularity DESC);
CREATE INDEX idx_anime_average_rating ON anime (average_rating DESC NULLS LAST);
CREATE INDEX idx_anime_is_adult ON anime (is_adult);
CREATE INDEX idx_anime_search_vector ON anime USING GIN (search_vector);
CREATE INDEX idx_anime_title_trgm ON anime USING GIN (title_en gin_trgm_ops, title_romaji gin_trgm_ops);
CREATE INDEX idx_anime_start_date ON anime (start_date DESC NULLS LAST);

CREATE TRIGGER trg_anime_updated_at
    BEFORE UPDATE ON anime
    FOR EACH ROW EXECUTE FUNCTION fn_set_updated_at();

-- ============================================================
-- TABLE: tv_series
-- Sourced from TMDB. tmdb_id is the stable external identifier.
-- ============================================================

CREATE TABLE tv_series (
    id                  UUID            NOT NULL DEFAULT gen_random_uuid(),
    tmdb_id             INTEGER         NOT NULL,
    title_en            TEXT            NULL,
    title_ar            TEXT            NULL,
    overview_en         TEXT            NULL,
    overview_ar         TEXT            NULL,
    poster_path         TEXT            NULL,       -- TMDB relative path, e.g. /abc123.jpg
    backdrop_path       TEXT            NULL,
    logo_path           TEXT            NULL,
    status              tv_series_status_enum NULL,
    first_air_date      DATE            NULL,
    last_air_date       DATE            NULL,
    number_of_seasons   SMALLINT        NULL CHECK (number_of_seasons >= 0),
    number_of_episodes  INTEGER         NULL CHECK (number_of_episodes >= 0),
    origin_country      CHAR(2)[]       NULL,       -- ISO 3166-1 alpha-2 array
    original_language   CHAR(5)         NULL,       -- BCP-47 language tag
    is_adult            BOOLEAN         NOT NULL DEFAULT FALSE,
    vote_average        DECIMAL(4,2)    NULL CHECK (vote_average BETWEEN 0 AND 10),
    vote_count          INTEGER         NOT NULL DEFAULT 0 CHECK (vote_count >= 0),
    popularity          DECIMAL(10,3)   NOT NULL DEFAULT 0 CHECK (popularity >= 0),
    trailer_url         TEXT            NULL,
    homepage            TEXT            NULL,
    tagline             TEXT            NULL,
    in_production       BOOLEAN         NOT NULL DEFAULT FALSE,
    -- Full-text search
    search_vector       TSVECTOR        GENERATED ALWAYS AS (
        setweight(to_tsvector('english', coalesce(title_en, '')), 'A') ||
        setweight(to_tsvector('arabic', coalesce(title_ar, '')), 'A') ||
        setweight(to_tsvector('english', coalesce(overview_en, '')), 'C')
    ) STORED,
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),

    CONSTRAINT pk_tv_series PRIMARY KEY (id),
    CONSTRAINT uq_tv_series_tmdb_id UNIQUE (tmdb_id),
    CONSTRAINT chk_tv_series_dates CHECK (
        last_air_date IS NULL OR first_air_date IS NULL OR last_air_date >= first_air_date
    )
);

CREATE INDEX idx_tv_series_tmdb_id ON tv_series (tmdb_id);
CREATE INDEX idx_tv_series_status ON tv_series (status);
CREATE INDEX idx_tv_series_first_air_date ON tv_series (first_air_date DESC NULLS LAST);
CREATE INDEX idx_tv_series_popularity ON tv_series (popularity DESC);
CREATE INDEX idx_tv_series_vote_average ON tv_series (vote_average DESC NULLS LAST);
CREATE INDEX idx_tv_series_is_adult ON tv_series (is_adult);
CREATE INDEX idx_tv_series_origin_country ON tv_series USING GIN (origin_country);
CREATE INDEX idx_tv_series_search_vector ON tv_series USING GIN (search_vector);
CREATE INDEX idx_tv_series_title_trgm ON tv_series USING GIN (title_en gin_trgm_ops);

CREATE TRIGGER trg_tv_series_updated_at
    BEFORE UPDATE ON tv_series
    FOR EACH ROW EXECUTE FUNCTION fn_set_updated_at();

-- ============================================================
-- TABLE: movies
-- Sourced from TMDB.
-- ============================================================

CREATE TABLE movies (
    id                  UUID            NOT NULL DEFAULT gen_random_uuid(),
    tmdb_id             INTEGER         NOT NULL,
    title_en            TEXT            NULL,
    title_ar            TEXT            NULL,
    overview_en         TEXT            NULL,
    overview_ar         TEXT            NULL,
    poster_path         TEXT            NULL,
    backdrop_path       TEXT            NULL,
    logo_path           TEXT            NULL,
    status              movie_status_enum NULL,
    release_date        DATE            NULL,
    runtime             SMALLINT        NULL CHECK (runtime > 0),          -- minutes
    budget              BIGINT          NULL CHECK (budget >= 0),          -- USD
    revenue             BIGINT          NULL CHECK (revenue >= 0),         -- USD
    original_language   CHAR(5)         NULL,
    origin_country      CHAR(2)[]       NULL,
    is_adult            BOOLEAN         NOT NULL DEFAULT FALSE,
    vote_average        DECIMAL(4,2)    NULL CHECK (vote_average BETWEEN 0 AND 10),
    vote_count          INTEGER         NOT NULL DEFAULT 0 CHECK (vote_count >= 0),
    popularity          DECIMAL(10,3)   NOT NULL DEFAULT 0 CHECK (popularity >= 0),
    trailer_url         TEXT            NULL,
    homepage            TEXT            NULL,
    tagline             TEXT            NULL,
    imdb_id             VARCHAR(20)     NULL,       -- e.g. tt0133093
    -- Full-text search
    search_vector       TSVECTOR        GENERATED ALWAYS AS (
        setweight(to_tsvector('english', coalesce(title_en, '')), 'A') ||
        setweight(to_tsvector('arabic', coalesce(title_ar, '')), 'A') ||
        setweight(to_tsvector('english', coalesce(overview_en, '')), 'C')
    ) STORED,
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),

    CONSTRAINT pk_movies PRIMARY KEY (id),
    CONSTRAINT uq_movies_tmdb_id UNIQUE (tmdb_id),
    CONSTRAINT uq_movies_imdb_id UNIQUE (imdb_id)
);

CREATE INDEX idx_movies_tmdb_id ON movies (tmdb_id);
CREATE INDEX idx_movies_status ON movies (status);
CREATE INDEX idx_movies_release_date ON movies (release_date DESC NULLS LAST);
CREATE INDEX idx_movies_popularity ON movies (popularity DESC);
CREATE INDEX idx_movies_vote_average ON movies (vote_average DESC NULLS LAST);
CREATE INDEX idx_movies_is_adult ON movies (is_adult);
CREATE INDEX idx_movies_runtime ON movies (runtime);
CREATE INDEX idx_movies_search_vector ON movies USING GIN (search_vector);
CREATE INDEX idx_movies_title_trgm ON movies USING GIN (title_en gin_trgm_ops);
CREATE INDEX idx_movies_origin_country ON movies USING GIN (origin_country);

CREATE TRIGGER trg_movies_updated_at
    BEFORE UPDATE ON movies
    FOR EACH ROW EXECUTE FUNCTION fn_set_updated_at();
```

---

## 8. Seasons & Episodes

```sql
-- ============================================================
-- TABLE: seasons
-- Belongs to tv_series. tmdb_id is TMDB's season_id.
-- Season 0 is the "Specials" season in TMDB convention.
-- ============================================================

CREATE TABLE seasons (
    id                  UUID            NOT NULL DEFAULT gen_random_uuid(),
    tv_series_id        UUID            NOT NULL,
    tmdb_id             INTEGER         NULL,       -- TMDB season_id, can be NULL for synthetic seasons
    season_number       SMALLINT        NOT NULL CHECK (season_number >= 0),
    name                TEXT            NULL,
    name_ar             TEXT            NULL,
    overview            TEXT            NULL,
    overview_ar         TEXT            NULL,
    poster_path         TEXT            NULL,
    air_date            DATE            NULL,
    episode_count       SMALLINT        NULL CHECK (episode_count >= 0),
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),

    CONSTRAINT pk_seasons PRIMARY KEY (id),
    CONSTRAINT uq_seasons_series_number UNIQUE (tv_series_id, season_number),
    CONSTRAINT uq_seasons_tmdb_id UNIQUE (tmdb_id),
    CONSTRAINT fk_seasons_tv_series FOREIGN KEY (tv_series_id)
        REFERENCES tv_series (id) ON DELETE CASCADE
);

CREATE INDEX idx_seasons_tv_series_id ON seasons (tv_series_id);
CREATE INDEX idx_seasons_air_date ON seasons (air_date DESC NULLS LAST);
CREATE INDEX idx_seasons_tmdb_id ON seasons (tmdb_id) WHERE tmdb_id IS NOT NULL;

CREATE TRIGGER trg_seasons_updated_at
    BEFORE UPDATE ON seasons
    FOR EACH ROW EXECUTE FUNCTION fn_set_updated_at();

-- ============================================================
-- TABLE: episodes
-- Belongs to a season (TV Series). episode_number is 1-based
-- within its season. tmdb_id is TMDB's episode_id.
-- ============================================================

CREATE TABLE episodes (
    id                  UUID            NOT NULL DEFAULT gen_random_uuid(),
    season_id           UUID            NOT NULL,
    tmdb_id             INTEGER         NULL,
    episode_number      SMALLINT        NOT NULL CHECK (episode_number >= 0),
    name                TEXT            NULL,
    name_ar             TEXT            NULL,
    overview            TEXT            NULL,
    overview_ar         TEXT            NULL,
    still_path          TEXT            NULL,
    air_date            DATE            NULL,
    runtime             SMALLINT        NULL CHECK (runtime > 0),      -- minutes
    vote_average        DECIMAL(4,2)    NULL CHECK (vote_average BETWEEN 0 AND 10),
    vote_count          INTEGER         NOT NULL DEFAULT 0 CHECK (vote_count >= 0),
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),

    CONSTRAINT pk_episodes PRIMARY KEY (id),
    CONSTRAINT uq_episodes_season_number UNIQUE (season_id, episode_number),
    CONSTRAINT uq_episodes_tmdb_id UNIQUE (tmdb_id),
    CONSTRAINT fk_episodes_season FOREIGN KEY (season_id)
        REFERENCES seasons (id) ON DELETE CASCADE
);

CREATE INDEX idx_episodes_season_id ON episodes (season_id);
CREATE INDEX idx_episodes_air_date ON episodes (air_date DESC NULLS LAST);
CREATE INDEX idx_episodes_tmdb_id ON episodes (tmdb_id) WHERE tmdb_id IS NOT NULL;

CREATE TRIGGER trg_episodes_updated_at
    BEFORE UPDATE ON episodes
    FOR EACH ROW EXECUTE FUNCTION fn_set_updated_at();

-- ============================================================
-- TABLE: anime_episodes
-- Separate from TV episodes because anime episodes are tied
-- to AniList episode data, not TMDB. anime_id references the
-- anime table. anilist_episode_id can be NULL for non-indexed
-- episodes (e.g., episode streaming platforms not on AniList).
-- ============================================================

CREATE TABLE anime_episodes (
    id                      UUID        NOT NULL DEFAULT gen_random_uuid(),
    anime_id                UUID        NOT NULL,
    anilist_episode_id      INTEGER     NULL,       -- AniList's internal episode ID when available
    episode_number          SMALLINT    NOT NULL CHECK (episode_number >= 1),
    title                   TEXT        NULL,
    title_ar                TEXT        NULL,
    thumbnail               TEXT        NULL,       -- Full URL to episode thumbnail
    air_date                DATE        NULL,
    duration                SMALLINT    NULL CHECK (duration > 0),   -- minutes
    overview                TEXT        NULL,
    is_filler               BOOLEAN     NOT NULL DEFAULT FALSE,
    created_at              TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT pk_anime_episodes PRIMARY KEY (id),
    CONSTRAINT uq_anime_episodes_number UNIQUE (anime_id, episode_number),
    CONSTRAINT uq_anime_episodes_anilist_id UNIQUE (anilist_episode_id),
    CONSTRAINT fk_anime_episodes_anime FOREIGN KEY (anime_id)
        REFERENCES anime (id) ON DELETE CASCADE
);

CREATE INDEX idx_anime_episodes_anime_id ON anime_episodes (anime_id);
CREATE INDEX idx_anime_episodes_air_date ON anime_episodes (air_date DESC NULLS LAST);
CREATE INDEX idx_anime_episodes_anilist_id ON anime_episodes (anilist_episode_id)
    WHERE anilist_episode_id IS NOT NULL;

CREATE TRIGGER trg_anime_episodes_updated_at
    BEFORE UPDATE ON anime_episodes
    FOR EACH ROW EXECUTE FUNCTION fn_set_updated_at();
```

---

## 9. People, Characters & Cast

```sql
-- ============================================================
-- TABLE: people
-- Actors, directors, writers sourced from TMDB.
-- known_for_department matches TMDB's department field.
-- ============================================================

CREATE TABLE people (
    id                      UUID        NOT NULL DEFAULT gen_random_uuid(),
    tmdb_id                 INTEGER     NOT NULL,
    name                    TEXT        NOT NULL,
    original_name           TEXT        NULL,
    profile_path            TEXT        NULL,
    biography               TEXT        NULL,
    birthday                DATE        NULL,
    deathday                DATE        NULL,
    gender                  SMALLINT    NULL CHECK (gender IN (0, 1, 2, 3)),  -- TMDB convention
    known_for_department    TEXT        NULL,       -- Acting, Directing, Writing, etc.
    place_of_birth          TEXT        NULL,
    popularity              DECIMAL(10,3) NOT NULL DEFAULT 0 CHECK (popularity >= 0),
    imdb_id                 VARCHAR(20) NULL,
    homepage                TEXT        NULL,
    created_at              TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CONSTRAINT pk_people PRIMARY KEY (id),
    CONSTRAINT uq_people_tmdb_id UNIQUE (tmdb_id),
    CONSTRAINT chk_people_dates CHECK (
        deathday IS NULL OR birthday IS NULL OR deathday >= birthday
    )
);

CREATE INDEX idx_people_tmdb_id ON people (tmdb_id);
CREATE INDEX idx_people_name_trgm ON people USING GIN (name gin_trgm_ops);
CREATE INDEX idx_people_popularity ON people (popularity DESC);

CREATE TRIGGER trg_people_updated_at
    BEFORE UPDATE ON people
    FOR EACH ROW EXECUTE FUNCTION fn_set_updated_at();

-- ============================================================
-- TABLE: characters
-- Represents fictional characters appearing in anime, TV, or
-- movies. For anime, characters come from AniList. For TV/
-- movies, characters are implied by the cast role.
-- media_type + media_id form a polymorphic FK (not enforced
-- by a single FK constraint — enforced at application layer
-- and via CHECK constraints per type).
-- ============================================================

CREATE TABLE characters (
    id                  UUID            NOT NULL DEFAULT gen_random_uuid(),
    name                TEXT            NOT NULL,
    name_native         TEXT            NULL,
    image               TEXT            NULL,
    description         TEXT            NULL,
    media_type          media_type_enum NOT NULL,
    media_id            UUID            NOT NULL,   -- references anime.id / tv_series.id / movies.id
    anilist_char_id     INTEGER         NULL,       -- AniList character ID for anime characters
    gender              TEXT            NULL,
    age                 TEXT            NULL,       -- stored as text (e.g., "17-18", "Unknown")
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),

    CONSTRAINT pk_characters PRIMARY KEY (id),
    CONSTRAINT uq_characters_anilist_id UNIQUE (anilist_char_id)
);

CREATE INDEX idx_characters_media ON characters (media_type, media_id);
CREATE INDEX idx_characters_anilist_char_id ON characters (anilist_char_id)
    WHERE anilist_char_id IS NOT NULL;
CREATE INDEX idx_characters_name_trgm ON characters USING GIN (name gin_trgm_ops);

CREATE TRIGGER trg_characters_updated_at
    BEFORE UPDATE ON characters
    FOR EACH ROW EXECUTE FUNCTION fn_set_updated_at();

-- ============================================================
-- TABLE: media_cast
-- Junction table connecting people to media (polymorphic).
-- "order" determines display order (lead actors first).
-- character_name is stored denormalized here because the same
-- actor can play different characters in different productions.
-- ============================================================

CREATE TABLE media_cast (
    id                  UUID            NOT NULL DEFAULT gen_random_uuid(),
    media_type          media_type_enum NOT NULL,
    media_id            UUID            NOT NULL,
    person_id           UUID            NOT NULL,
    character_name      TEXT            NULL,
    character_id        UUID            NULL,       -- optional FK to characters table
    department          TEXT            NOT NULL DEFAULT 'Acting',
    job                 TEXT            NULL,       -- e.g., Director, Writer, Producer
    cast_order          SMALLINT        NULL CHECK (cast_order >= 0),
    credit_id           VARCHAR(100)    NULL,       -- TMDB credit_id for deduplication
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),

    CONSTRAINT pk_media_cast PRIMARY KEY (id),
    CONSTRAINT uq_media_cast_credit UNIQUE (credit_id),
    CONSTRAINT fk_media_cast_person FOREIGN KEY (person_id)
        REFERENCES people (id) ON DELETE CASCADE,
    CONSTRAINT fk_media_cast_character FOREIGN KEY (character_id)
        REFERENCES characters (id) ON DELETE SET NULL
);

CREATE INDEX idx_media_cast_media ON media_cast (media_type, media_id);
CREATE INDEX idx_media_cast_person_id ON media_cast (person_id);
CREATE INDEX idx_media_cast_order ON media_cast (media_type, media_id, cast_order ASC NULLS LAST);
```

---

## 10. Studios & Networks

```sql
-- ============================================================
-- TABLE: studios
-- Anime production studios sourced from AniList.
-- Some studios are also animation studios for TMDB content
-- (production_companies), differentiated by which ID is set.
-- ============================================================

CREATE TABLE studios (
    id                  UUID            NOT NULL DEFAULT gen_random_uuid(),
    name                TEXT            NOT NULL,
    country             CHAR(2)         NULL,       -- ISO 3166-1 alpha-2
    logo_url            TEXT            NULL,
    anilist_id          INTEGER         NULL,
    tmdb_id             INTEGER         NULL,       -- for production companies on TMDB
    is_animation_studio BOOLEAN         NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),

    CONSTRAINT pk_studios PRIMARY KEY (id),
    CONSTRAINT uq_studios_anilist_id UNIQUE (anilist_id),
    CONSTRAINT uq_studios_tmdb_id UNIQUE (tmdb_id),
    CONSTRAINT chk_studios_has_source CHECK (anilist_id IS NOT NULL OR tmdb_id IS NOT NULL)
);

CREATE INDEX idx_studios_anilist_id ON studios (anilist_id) WHERE anilist_id IS NOT NULL;
CREATE INDEX idx_studios_tmdb_id ON studios (tmdb_id) WHERE tmdb_id IS NOT NULL;
CREATE INDEX idx_studios_name_trgm ON studios USING GIN (name gin_trgm_ops);

CREATE TRIGGER trg_studios_updated_at
    BEFORE UPDATE ON studios
    FOR EACH ROW EXECUTE FUNCTION fn_set_updated_at();

-- ============================================================
-- TABLE: anime_studios
-- Junction: which studios produced which anime.
-- is_main distinguishes the main studio from sub-studios.
-- ============================================================

CREATE TABLE anime_studios (
    anime_id            UUID            NOT NULL,
    studio_id           UUID            NOT NULL,
    is_main             BOOLEAN         NOT NULL DEFAULT FALSE,

    CONSTRAINT pk_anime_studios PRIMARY KEY (anime_id, studio_id),
    CONSTRAINT fk_anime_studios_anime FOREIGN KEY (anime_id)
        REFERENCES anime (id) ON DELETE CASCADE,
    CONSTRAINT fk_anime_studios_studio FOREIGN KEY (studio_id)
        REFERENCES studios (id) ON DELETE CASCADE
);

CREATE INDEX idx_anime_studios_studio_id ON anime_studios (studio_id);

-- ============================================================
-- TABLE: networks
-- TV broadcast/streaming networks sourced from TMDB.
-- ============================================================

CREATE TABLE networks (
    id                  UUID            NOT NULL DEFAULT gen_random_uuid(),
    tmdb_id             INTEGER         NOT NULL,
    name                TEXT            NOT NULL,
    logo_path           TEXT            NULL,
    origin_country      CHAR(2)         NULL,
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),

    CONSTRAINT pk_networks PRIMARY KEY (id),
    CONSTRAINT uq_networks_tmdb_id UNIQUE (tmdb_id)
);

CREATE INDEX idx_networks_tmdb_id ON networks (tmdb_id);

-- ============================================================
-- TABLE: tv_series_networks
-- Junction: which networks air which TV series.
-- ============================================================

CREATE TABLE tv_series_networks (
    tv_series_id        UUID            NOT NULL,
    network_id          UUID            NOT NULL,

    CONSTRAINT pk_tv_series_networks PRIMARY KEY (tv_series_id, network_id),
    CONSTRAINT fk_tv_series_networks_series FOREIGN KEY (tv_series_id)
        REFERENCES tv_series (id) ON DELETE CASCADE,
    CONSTRAINT fk_tv_series_networks_network FOREIGN KEY (network_id)
        REFERENCES networks (id) ON DELETE CASCADE
);

CREATE INDEX idx_tv_series_networks_network_id ON tv_series_networks (network_id);
```

---

## 11. Genres

```sql
-- ============================================================
-- TABLE: genres
-- Unified genre table covering both AniList and TMDB genres.
-- AniList uses string names; TMDB uses integer IDs with names.
-- tmdb_id is NULL for anime-only genres (e.g., "Mecha").
-- anilist_name is NULL for movie/TV-only genres.
-- ============================================================

CREATE TABLE genres (
    id                  UUID            NOT NULL DEFAULT gen_random_uuid(),
    name_en             VARCHAR(100)    NOT NULL,
    name_ar             VARCHAR(100)    NULL,
    tmdb_id             INTEGER         NULL,       -- TMDB genre_id (shared for movie & TV)
    anilist_name        VARCHAR(100)    NULL,       -- AniList genre string identifier
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),

    CONSTRAINT pk_genres PRIMARY KEY (id),
    CONSTRAINT uq_genres_tmdb_id UNIQUE (tmdb_id),
    CONSTRAINT uq_genres_anilist_name UNIQUE (anilist_name),
    CONSTRAINT uq_genres_name_en UNIQUE (name_en)
);

CREATE INDEX idx_genres_name_en ON genres (name_en);

-- ============================================================
-- TABLE: media_genres
-- Polymorphic junction: any media type can have multiple genres.
-- ============================================================

CREATE TABLE media_genres (
    media_type          media_type_enum NOT NULL,
    media_id            UUID            NOT NULL,
    genre_id            UUID            NOT NULL,

    CONSTRAINT pk_media_genres PRIMARY KEY (media_type, media_id, genre_id),
    CONSTRAINT fk_media_genres_genre FOREIGN KEY (genre_id)
        REFERENCES genres (id) ON DELETE CASCADE
);

CREATE INDEX idx_media_genres_genre_id ON media_genres (genre_id);
CREATE INDEX idx_media_genres_media ON media_genres (media_type, media_id);
```

---

## 12. User Media Tracking

```sql
-- ============================================================
-- TABLE: user_media_entries
-- Core tracking record. One entry per (user, media_type, media).
-- rating is on a 1.0–10.0 scale with 0.5 increments.
-- rewatch_count counts completed rewatches (not including the
-- first watch). notes is user's private journal-style note.
-- ============================================================

CREATE TABLE user_media_entries (
    id                  UUID                    NOT NULL DEFAULT gen_random_uuid(),
    user_id             UUID                    NOT NULL,
    media_type          media_type_enum         NOT NULL,
    media_id            UUID                    NOT NULL,
    status              user_media_status_enum  NOT NULL,
    rating              DECIMAL(3,1)            NULL
        CHECK (rating IS NULL OR (rating >= 1.0 AND rating <= 10.0 AND (rating * 10) % 5 = 0)),
    rewatch_count       SMALLINT                NOT NULL DEFAULT 0 CHECK (rewatch_count >= 0),
    notes               TEXT                    NULL,
    started_at          TIMESTAMPTZ             NULL,
    completed_at        TIMESTAMPTZ             NULL,
    created_at          TIMESTAMPTZ             NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ             NOT NULL DEFAULT NOW(),

    CONSTRAINT pk_user_media_entries PRIMARY KEY (id),
    CONSTRAINT uq_user_media_entries_entry UNIQUE (user_id, media_type, media_id),
    CONSTRAINT fk_user_media_entries_user FOREIGN KEY (user_id)
        REFERENCES users (id) ON DELETE CASCADE,
    CONSTRAINT chk_user_media_entries_dates CHECK (
        completed_at IS NULL OR started_at IS NULL OR completed_at >= started_at
    ),
    CONSTRAINT chk_user_media_entries_rewatch_completed CHECK (
        rewatch_count = 0 OR status IN ('completed', 'rewatching')
    )
);

CREATE INDEX idx_user_media_entries_user_id ON user_media_entries (user_id);
CREATE INDEX idx_user_media_entries_media ON user_media_entries (media_type, media_id);
CREATE INDEX idx_user_media_entries_status ON user_media_entries (user_id, status);
CREATE INDEX idx_user_media_entries_rating ON user_media_entries (user_id, rating DESC NULLS LAST)
    WHERE rating IS NOT NULL;
CREATE INDEX idx_user_media_entries_updated ON user_media_entries (user_id, updated_at DESC);
CREATE INDEX idx_user_media_entries_completed_at ON user_media_entries (user_id, completed_at DESC)
    WHERE completed_at IS NOT NULL;

CREATE TRIGGER trg_user_media_entries_updated_at
    BEFORE UPDATE ON user_media_entries
    FOR EACH ROW EXECUTE FUNCTION fn_set_updated_at();

-- ============================================================
-- TABLE: user_episode_progress
-- Tracks per-episode watch events. A user can re-watch episodes,
-- so watched_at is part of the natural key for rewatches.
-- For simplicity, we use a surrogate PK and allow multiple rows
-- per (user, episode) — the application uses the latest row.
-- rating here is per-episode (optional, 1.0–10.0).
-- ============================================================

CREATE TABLE user_episode_progress (
    id                  UUID            NOT NULL DEFAULT gen_random_uuid(),
    user_id             UUID            NOT NULL,
    media_type          media_type_enum NOT NULL,   -- 'anime' or 'tv_series'
    episode_id          UUID            NOT NULL,   -- references anime_episodes.id or episodes.id
    watched_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    rating              DECIMAL(3,1)    NULL
        CHECK (rating IS NULL OR (rating >= 1.0 AND rating <= 10.0 AND (rating * 10) % 5 = 0)),

    CONSTRAINT pk_user_episode_progress PRIMARY KEY (id),
    CONSTRAINT fk_user_episode_progress_user FOREIGN KEY (user_id)
        REFERENCES users (id) ON DELETE CASCADE,
    CONSTRAINT chk_user_episode_progress_media_type CHECK (
        media_type IN ('anime', 'tv_series')
    )
);

CREATE INDEX idx_user_episode_progress_user_id ON user_episode_progress (user_id);
CREATE INDEX idx_user_episode_progress_episode ON user_episode_progress (media_type, episode_id);
CREATE INDEX idx_user_episode_progress_user_episode ON user_episode_progress (user_id, media_type, episode_id);
CREATE INDEX idx_user_episode_progress_watched_at ON user_episode_progress (user_id, watched_at DESC);

-- Unique index to prevent duplicate watch entries within 5 seconds
-- (application enforces idempotency, DB enforces hard uniqueness per day)
CREATE UNIQUE INDEX idx_user_episode_progress_unique_day ON user_episode_progress
    (user_id, media_type, episode_id, DATE_TRUNC('day', watched_at));
```

---

## 13. Reviews, Comments & Likes

```sql
-- ============================================================
-- TABLE: reviews
-- Users can write one review per (media_type, media_id).
-- Soft-deleted via deleted_at.
-- image_url_1, image_url_2 allow up to two optional images.
-- ============================================================

CREATE TABLE reviews (
    id                  UUID            NOT NULL DEFAULT gen_random_uuid(),
    user_id             UUID            NOT NULL,
    media_type          media_type_enum NOT NULL,
    media_id            UUID            NOT NULL,
    body                TEXT            NOT NULL,
    rating              DECIMAL(3,1)    NULL
        CHECK (rating IS NULL OR (rating >= 1.0 AND rating <= 10.0 AND (rating * 10) % 5 = 0)),
    is_spoiler          BOOLEAN         NOT NULL DEFAULT FALSE,
    image_url_1         TEXT            NULL,
    image_url_2         TEXT            NULL,
    like_count          INTEGER         NOT NULL DEFAULT 0 CHECK (like_count >= 0),  -- denormalized
    comment_count       INTEGER         NOT NULL DEFAULT 0 CHECK (comment_count >= 0), -- denormalized
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    deleted_at          TIMESTAMPTZ     NULL,

    CONSTRAINT pk_reviews PRIMARY KEY (id),
    CONSTRAINT uq_reviews_user_media UNIQUE (user_id, media_type, media_id)
        DEFERRABLE INITIALLY DEFERRED,   -- allows same user to delete and re-review same media
    CONSTRAINT fk_reviews_user FOREIGN KEY (user_id)
        REFERENCES users (id) ON DELETE CASCADE,
    CONSTRAINT chk_reviews_body_length CHECK (char_length(body) >= 10)
);

-- Partial unique index that ignores soft-deleted reviews
CREATE UNIQUE INDEX idx_reviews_active_user_media ON reviews
    (user_id, media_type, media_id) WHERE deleted_at IS NULL;

CREATE INDEX idx_reviews_media ON reviews (media_type, media_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_reviews_user_id ON reviews (user_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_reviews_created_at ON reviews (media_type, media_id, created_at DESC)
    WHERE deleted_at IS NULL;
CREATE INDEX idx_reviews_rating ON reviews (media_type, media_id, rating DESC NULLS LAST)
    WHERE deleted_at IS NULL AND rating IS NOT NULL;
CREATE INDEX idx_reviews_like_count ON reviews (media_type, media_id, like_count DESC)
    WHERE deleted_at IS NULL;
CREATE INDEX idx_reviews_deleted_at ON reviews (deleted_at) WHERE deleted_at IS NOT NULL;

CREATE TRIGGER trg_reviews_updated_at
    BEFORE UPDATE ON reviews
    FOR EACH ROW EXECUTE FUNCTION fn_set_updated_at();

-- ============================================================
-- TABLE: review_likes
-- Many-to-many between users and reviews.
-- On like: increment reviews.like_count via trigger.
-- On unlike: decrement reviews.like_count via trigger.
-- ============================================================

CREATE TABLE review_likes (
    review_id           UUID            NOT NULL,
    user_id             UUID            NOT NULL,
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),

    CONSTRAINT pk_review_likes PRIMARY KEY (review_id, user_id),
    CONSTRAINT fk_review_likes_review FOREIGN KEY (review_id)
        REFERENCES reviews (id) ON DELETE CASCADE,
    CONSTRAINT fk_review_likes_user FOREIGN KEY (user_id)
        REFERENCES users (id) ON DELETE CASCADE
);

CREATE INDEX idx_review_likes_user_id ON review_likes (user_id);
CREATE INDEX idx_review_likes_review_id ON review_likes (review_id);
CREATE INDEX idx_review_likes_created_at ON review_likes (review_id, created_at DESC);

-- Trigger: keep reviews.like_count in sync
CREATE OR REPLACE FUNCTION fn_update_review_like_count()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        UPDATE reviews SET like_count = like_count + 1 WHERE id = NEW.review_id;
    ELSIF TG_OP = 'DELETE' THEN
        UPDATE reviews SET like_count = GREATEST(like_count - 1, 0) WHERE id = OLD.review_id;
    END IF;
    RETURN NULL;
END;
$$;

CREATE TRIGGER trg_review_likes_count
    AFTER INSERT OR DELETE ON review_likes
    FOR EACH ROW EXECUTE FUNCTION fn_update_review_like_count();

-- ============================================================
-- TABLE: comments
-- Threaded comments on reviews. parent_comment_id supports
-- one level of nesting (replies to comments). To prevent
-- deeper nesting, a check ensures parent comments themselves
-- have no parent (enforced at application layer; DB stores
-- the tree structure without depth limit for flexibility).
-- Soft-deleted via deleted_at.
-- ============================================================

CREATE TABLE comments (
    id                  UUID            NOT NULL DEFAULT gen_random_uuid(),
    review_id           UUID            NOT NULL,
    parent_comment_id   UUID            NULL,
    user_id             UUID            NOT NULL,
    body                TEXT            NOT NULL,
    like_count          INTEGER         NOT NULL DEFAULT 0 CHECK (like_count >= 0),
    created_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    deleted_at          TIMESTAMPTZ     NULL,

    CONSTRAINT pk_comments PRIMARY KEY (id),
    CONSTRAINT fk_comments_review FOREIGN KEY (review_id)
        REFERENCES reviews (id) ON DELETE CASCADE,
    CONSTRAINT fk_comments_parent FOREIGN KEY (parent_comment_id)
        REFERENCES comments (id) ON DELETE CASCADE,
    CONSTRAINT fk_comments_user FOREIGN KEY (user_id)
        REFERENCES users (id) ON DELETE CASCADE,
    CONSTRAINT chk_comments_body_length CHECK (char_length(body) >= 1),
    CONSTRAINT chk_comments_not_self_parent CHECK (id <> parent_comment_id)
);

CREATE INDEX idx_comments_review_id ON comments (review_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_comments_parent_id ON comments (parent_comment_id) WHERE parent_comment_id IS NOT NULL;
CREATE INDEX idx_comments_user_id ON comments (user_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_comments_created_at ON comments (review_id, created_at ASC) WHERE deleted_at IS NULL;

CREATE TRIGGER trg_comments_updated_at
    BEFORE UPDATE ON comments
    FOR EACH ROW EXECUTE FUNCTION fn_set_updated_at();

-- Trigger: keep reviews.comment_count in sync
CREATE OR REPLACE FUNCTION fn_update_review_comment_count()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    IF TG_OP = 'INSERT' AND NEW.deleted_at IS NULL THEN
        UPDATE reviews SET comment_count = comment_count + 1 WHERE id = NEW.review_id;
    ELSIF TG_OP = 'UPDATE' THEN
        IF OLD.deleted_at IS NULL AND NEW.deleted_at IS NOT NULL THEN
            UPDATE reviews SET comment_count = GREATEST(comment_count - 1, 0)
                WHERE id = NEW.review_id;
        ELSIF OLD.deleted_at IS NOT NULL AND NEW.deleted_at IS NULL THEN
            UPDATE reviews SET comment_count = comment_count + 1 WHERE id = NEW.review_id;
        END IF;
    END IF;
    RETURN NULL;
END;
$$;

CREATE TRIGGER trg_comments_count
    AFTER INSERT OR UPDATE OF deleted_at ON comments
    FOR EACH ROW EXECUTE FUNCTION fn_update_review_comment_count();
```

---

## 14. Lists & Favorites

```sql
-- ============================================================
-- TABLE: lists
-- User-curated lists of media. slug is auto-generated from
-- title and must be unique per user for shareable URLs.
-- sort_order is the display order among a user's lists.
-- ============================================================

CREATE TABLE lists (
    id                  UUID                NOT NULL DEFAULT gen_random_uuid(),
    user_id             UUID                NOT NULL,
    title               VARCHAR(200)        NOT NULL,
    slug                VARCHAR(220)        NOT NULL,
    description         TEXT                NULL,
    banner_url          TEXT                NULL,
    visibility          list_visibility_enum NOT NULL DEFAULT 'public',
    sort_order          INTEGER             NOT NULL DEFAULT 0,
    item_count          INTEGER             NOT NULL DEFAULT 0 CHECK (item_count >= 0),  -- denormalized
    created_at          TIMESTAMPTZ         NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ         NOT NULL DEFAULT NOW(),

    CONSTRAINT pk_lists PRIMARY KEY (id),
    CONSTRAINT uq_lists_user_slug UNIQUE (user_id, slug),
    CONSTRAINT fk_lists_user FOREIGN KEY (user_id)
        REFERENCES users (id) ON DELETE CASCADE,
    CONSTRAINT chk_lists_title_length CHECK (char_length(title) >= 1)
);

CREATE INDEX idx_lists_user_id ON lists (user_id);
CREATE INDEX idx_lists_visibility ON lists (visibility, updated_at DESC)
    WHERE visibility = 'public';
CREATE INDEX idx_lists_sort_order ON lists (user_id, sort_order ASC);

CREATE TRIGGER trg_lists_updated_at
    BEFORE UPDATE ON lists
    FOR EACH ROW EXECUTE FUNCTION fn_set_updated_at();

-- ============================================================
-- TABLE: list_items
-- Items inside a list. position determines the user-defined
-- ordering within the list (1-based).
-- ============================================================

CREATE TABLE list_items (
    id                  UUID            NOT NULL DEFAULT gen_random_uuid(),
    list_id             UUID            NOT NULL,
    media_type          media_type_enum NOT NULL,
    media_id            UUID            NOT NULL,
    position            INTEGER         NOT NULL CHECK (position >= 1),
    added_at            TIMESTAMPTZ     NOT NULL DEFAULT NOW(),

    CONSTRAINT pk_list_items PRIMARY KEY (id),
    CONSTRAINT uq_list_items_media UNIQUE (list_id, media_type, media_id),
    CONSTRAINT fk_list_items_list FOREIGN KEY (list_id)
        REFERENCES lists (id) ON DELETE CASCADE
);

CREATE INDEX idx_list_items_list_id ON list_items (list_id, position ASC);
CREATE INDEX idx_list_items_media ON list_items (media_type, media_id);

-- Trigger: keep lists.item_count in sync
CREATE OR REPLACE FUNCTION fn_update_list_item_count()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        UPDATE lists SET item_count = item_count + 1 WHERE id = NEW.list_id;
    ELSIF TG_OP = 'DELETE' THEN
        UPDATE lists SET item_count = GREATEST(item_count - 1, 0) WHERE id = OLD.list_id;
    END IF;
    RETURN NULL;
END;
$$;

CREATE TRIGGER trg_list_items_count
    AFTER INSERT OR DELETE ON list_items
    FOR EACH ROW EXECUTE FUNCTION fn_update_list_item_count();

-- ============================================================
-- TABLE: favorites
-- Pinned media on a user's profile. Limited to TV Series and
-- Movies (anime is tracked separately via user_media_entries
-- with status=completed). position controls display order.
-- ============================================================

CREATE TABLE favorites (
    id                      UUID                        NOT NULL DEFAULT gen_random_uuid(),
    user_id                 UUID                        NOT NULL,
    media_type              favorites_media_type_enum   NOT NULL,
    media_id                UUID                        NOT NULL,
    position                SMALLINT                    NOT NULL CHECK (position >= 1 AND position <= 20),
    created_at              TIMESTAMPTZ                 NOT NULL DEFAULT NOW(),

    CONSTRAINT pk_favorites PRIMARY KEY (id),
    CONSTRAINT uq_favorites_user_media UNIQUE (user_id, media_type, media_id),
    CONSTRAINT uq_favorites_position UNIQUE (user_id, media_type, position),
    CONSTRAINT fk_favorites_user FOREIGN KEY (user_id)
        REFERENCES users (id) ON DELETE CASCADE
);

CREATE INDEX idx_favorites_user_id ON favorites (user_id);
CREATE INDEX idx_favorites_media ON favorites (media_type, media_id);
CREATE INDEX idx_favorites_position ON favorites (user_id, media_type, position ASC);
```

---

## 15. Notifications & Activity Feed

```sql
-- ============================================================
-- TABLE: notifications (PARTITIONED)
-- Partitioned by created_at month. Old partitions are dropped
-- after 90 days. data JSONB holds type-specific payload:
--   review_like:      { review_id, liker_user_id, liker_username }
--   comment_reply:    { comment_id, review_id, commenter_user_id }
--   new_follower:     { follower_user_id, follower_username }
--   new_episode:      { media_type, media_id, episode_id, episode_number }
--   new_season:       { tv_series_id, season_number }
--   mention:          { comment_id, review_id, mentioner_user_id }
-- ============================================================

CREATE TABLE notifications (
    id                  UUID                    NOT NULL DEFAULT gen_random_uuid(),
    user_id             UUID                    NOT NULL,
    type                notification_type_enum  NOT NULL,
    data                JSONB                   NOT NULL DEFAULT '{}',
    read_at             TIMESTAMPTZ             NULL,
    created_at          TIMESTAMPTZ             NOT NULL DEFAULT NOW(),

    CONSTRAINT pk_notifications PRIMARY KEY (id, created_at),
    CONSTRAINT fk_notifications_user FOREIGN KEY (user_id)
        REFERENCES users (id) ON DELETE CASCADE
) PARTITION BY RANGE (created_at);

-- Create monthly partitions (managed by pg_partman in production)
CREATE TABLE notifications_y2025m01 PARTITION OF notifications
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
CREATE TABLE notifications_y2025m02 PARTITION OF notifications
    FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');
-- ... additional partitions created automatically by pg_partman

-- Indexes created on the parent; inherited by all partitions
CREATE INDEX idx_notifications_user_unread ON notifications
    (user_id, created_at DESC) WHERE read_at IS NULL;
CREATE INDEX idx_notifications_user_all ON notifications
    (user_id, created_at DESC);
CREATE INDEX idx_notifications_data_gin ON notifications USING GIN (data);

-- ============================================================
-- TABLE: activities (PARTITIONED)
-- User activity feed. Partitioned by month. Retained 30 days.
-- Followers see activities from accounts they follow.
-- data JSONB holds event-specific details:
--   watched_episode:    { media_type, media_id, episode_id, episode_number, media_title }
--   completed_movie:    { media_id, media_title, rating }
--   completed_series:   { media_id, media_title, rating }
--   completed_anime:    { media_id, media_title, rating, episode_count }
--   rating:             { media_type, media_id, media_title, rating }
--   review:             { review_id, media_type, media_id, media_title, rating }
--   new_list:           { list_id, list_title }
--   added_to_list:      { list_id, list_title, media_type, media_id, media_title }
--   started_watching:   { media_type, media_id, media_title }
--   dropped:            { media_type, media_id, media_title }
--   rewatching:         { media_type, media_id, media_title, rewatch_count }
-- ============================================================

CREATE TABLE activities (
    id                  UUID                NOT NULL DEFAULT gen_random_uuid(),
    user_id             UUID                NOT NULL,
    type                activity_type_enum  NOT NULL,
    data                JSONB               NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ         NOT NULL DEFAULT NOW(),

    CONSTRAINT pk_activities PRIMARY KEY (id, created_at),
    CONSTRAINT fk_activities_user FOREIGN KEY (user_id)
        REFERENCES users (id) ON DELETE CASCADE
) PARTITION BY RANGE (created_at);

CREATE TABLE activities_y2025m01 PARTITION OF activities
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
-- ... additional partitions managed by pg_partman

CREATE INDEX idx_activities_user_id ON activities (user_id, created_at DESC);
CREATE INDEX idx_activities_type ON activities (user_id, type, created_at DESC);
CREATE INDEX idx_activities_data_gin ON activities USING GIN (data);
```

---

## 16. Import / Sync System

```sql
-- ============================================================
-- TABLE: sync_jobs
-- One row per (user, provider) sync configuration.
-- Tracks scheduling and error state for background import jobs.
-- ============================================================

CREATE TABLE sync_jobs (
    id                  UUID                    NOT NULL DEFAULT gen_random_uuid(),
    user_id             UUID                    NOT NULL,
    provider            sync_provider_enum      NOT NULL,
    status              sync_job_status_enum    NOT NULL DEFAULT 'pending',
    last_synced_at      TIMESTAMPTZ             NULL,
    next_sync_at        TIMESTAMPTZ             NULL,
    error_message       TEXT                    NULL,
    total_imported      INTEGER                 NOT NULL DEFAULT 0 CHECK (total_imported >= 0),
    total_updated       INTEGER                 NOT NULL DEFAULT 0 CHECK (total_updated >= 0),
    total_conflicts     INTEGER                 NOT NULL DEFAULT 0 CHECK (total_conflicts >= 0),
    created_at          TIMESTAMPTZ             NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ             NOT NULL DEFAULT NOW(),

    CONSTRAINT pk_sync_jobs PRIMARY KEY (id),
    CONSTRAINT uq_sync_jobs_user_provider UNIQUE (user_id, provider),
    CONSTRAINT fk_sync_jobs_user FOREIGN KEY (user_id)
        REFERENCES users (id) ON DELETE CASCADE
);

CREATE INDEX idx_sync_jobs_user_id ON sync_jobs (user_id);
CREATE INDEX idx_sync_jobs_next_sync ON sync_jobs (next_sync_at ASC NULLS LAST)
    WHERE status IN ('pending', 'completed', 'partial');
CREATE INDEX idx_sync_jobs_status ON sync_jobs (status, next_sync_at ASC);

CREATE TRIGGER trg_sync_jobs_updated_at
    BEFORE UPDATE ON sync_jobs
    FOR EACH ROW EXECUTE FUNCTION fn_set_updated_at();

-- ============================================================
-- TABLE: sync_conflicts
-- When import data conflicts with existing local data,
-- a conflict record is created for manual user resolution.
-- local_data and remote_data store the competing states.
-- ============================================================

CREATE TABLE sync_conflicts (
    id                  UUID                    NOT NULL DEFAULT gen_random_uuid(),
    sync_job_id         UUID                    NOT NULL,
    media_type          media_type_enum         NOT NULL,
    media_id            UUID                    NOT NULL,
    local_data          JSONB                   NOT NULL,
    remote_data         JSONB                   NOT NULL,
    resolved_at         TIMESTAMPTZ             NULL,
    resolution          sync_resolution_enum    NULL,
    created_at          TIMESTAMPTZ             NOT NULL DEFAULT NOW(),

    CONSTRAINT pk_sync_conflicts PRIMARY KEY (id),
    CONSTRAINT fk_sync_conflicts_job FOREIGN KEY (sync_job_id)
        REFERENCES sync_jobs (id) ON DELETE CASCADE,
    CONSTRAINT chk_sync_conflicts_resolution CHECK (
        (resolved_at IS NULL AND resolution IS NULL) OR
        (resolved_at IS NOT NULL AND resolution IS NOT NULL)
    )
);

CREATE INDEX idx_sync_conflicts_sync_job_id ON sync_conflicts (sync_job_id);
CREATE INDEX idx_sync_conflicts_unresolved ON sync_conflicts (sync_job_id, created_at ASC)
    WHERE resolved_at IS NULL;
CREATE INDEX idx_sync_conflicts_media ON sync_conflicts (media_type, media_id);
```

---

## 17. Statistics & Cached Aggregates

```sql
-- ============================================================
-- TABLE: user_stats
-- Denormalized summary of a user's watch activity.
-- Updated by triggers on user_episode_progress INSERT and
-- user_media_entries UPDATE. Redis mirrors this with a
-- 5-minute TTL. Rebuilt from scratch during sync jobs.
-- ============================================================

CREATE TABLE user_stats (
    user_id                 UUID            NOT NULL,
    -- Episode counts
    total_anime_episodes    INTEGER         NOT NULL DEFAULT 0 CHECK (total_anime_episodes >= 0),
    total_tv_episodes       INTEGER         NOT NULL DEFAULT 0 CHECK (total_tv_episodes >= 0),
    -- Movie/series completion counts
    total_movies_watched    INTEGER         NOT NULL DEFAULT 0 CHECK (total_movies_watched >= 0),
    total_series_completed  INTEGER         NOT NULL DEFAULT 0 CHECK (total_series_completed >= 0),
    total_anime_completed   INTEGER         NOT NULL DEFAULT 0 CHECK (total_anime_completed >= 0),
    -- Watch time (in minutes)
    total_watch_time_minutes BIGINT         NOT NULL DEFAULT 0 CHECK (total_watch_time_minutes >= 0),
    -- Social counts (updated by triggers on follows/reviews)
    follower_count          INTEGER         NOT NULL DEFAULT 0 CHECK (follower_count >= 0),
    following_count         INTEGER         NOT NULL DEFAULT 0 CHECK (following_count >= 0),
    review_count            INTEGER         NOT NULL DEFAULT 0 CHECK (review_count >= 0),
    list_count              INTEGER         NOT NULL DEFAULT 0 CHECK (list_count >= 0),
    -- Mean ratings
    mean_anime_rating       DECIMAL(4,2)    NULL,
    mean_movie_rating       DECIMAL(4,2)    NULL,
    mean_tv_rating          DECIMAL(4,2)    NULL,
    -- Timestamps
    updated_at              TIMESTAMPTZ     NOT NULL DEFAULT NOW(),

    CONSTRAINT pk_user_stats PRIMARY KEY (user_id),
    CONSTRAINT fk_user_stats_user FOREIGN KEY (user_id)
        REFERENCES users (id) ON DELETE CASCADE
);

-- Auto-create stats row on user insert
CREATE OR REPLACE FUNCTION fn_create_user_stats()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    INSERT INTO user_stats (user_id) VALUES (NEW.id);
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_users_create_stats
    AFTER INSERT ON users
    FOR EACH ROW EXECUTE FUNCTION fn_create_user_stats();

-- Trigger: update stats when episode is marked watched
CREATE OR REPLACE FUNCTION fn_update_stats_on_episode_watch()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
DECLARE
    v_duration SMALLINT;
BEGIN
    IF NEW.media_type = 'anime' THEN
        SELECT duration INTO v_duration
        FROM anime_episodes WHERE id = NEW.episode_id;

        UPDATE user_stats
        SET
            total_anime_episodes = total_anime_episodes + 1,
            total_watch_time_minutes = total_watch_time_minutes + COALESCE(v_duration, 24),
            updated_at = NOW()
        WHERE user_id = NEW.user_id;

    ELSIF NEW.media_type = 'tv_series' THEN
        SELECT runtime INTO v_duration
        FROM episodes WHERE id = NEW.episode_id;

        UPDATE user_stats
        SET
            total_tv_episodes = total_tv_episodes + 1,
            total_watch_time_minutes = total_watch_time_minutes + COALESCE(v_duration, 45),
            updated_at = NOW()
        WHERE user_id = NEW.user_id;
    END IF;

    RETURN NULL;
END;
$$;

CREATE TRIGGER trg_episode_watch_stats
    AFTER INSERT ON user_episode_progress
    FOR EACH ROW EXECUTE FUNCTION fn_update_stats_on_episode_watch();

-- Trigger: update stats when movie is marked completed
CREATE OR REPLACE FUNCTION fn_update_stats_on_media_entry()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
DECLARE
    v_runtime INTEGER;
BEGIN
    -- Handle movie completions
    IF NEW.media_type = 'movie' AND NEW.status = 'completed' THEN
        IF OLD.status IS DISTINCT FROM 'completed' THEN
            SELECT runtime INTO v_runtime FROM movies WHERE id = NEW.media_id;
            UPDATE user_stats
            SET
                total_movies_watched = total_movies_watched + 1,
                total_watch_time_minutes = total_watch_time_minutes + COALESCE(v_runtime, 0),
                updated_at = NOW()
            WHERE user_id = NEW.user_id;
        END IF;
    END IF;

    -- Handle series completions
    IF NEW.media_type = 'tv_series' AND NEW.status = 'completed' THEN
        IF OLD.status IS DISTINCT FROM 'completed' THEN
            UPDATE user_stats
            SET total_series_completed = total_series_completed + 1, updated_at = NOW()
            WHERE user_id = NEW.user_id;
        END IF;
    END IF;

    -- Handle anime completions
    IF NEW.media_type = 'anime' AND NEW.status = 'completed' THEN
        IF OLD.status IS DISTINCT FROM 'completed' THEN
            UPDATE user_stats
            SET total_anime_completed = total_anime_completed + 1, updated_at = NOW()
            WHERE user_id = NEW.user_id;
        END IF;
    END IF;

    RETURN NULL;
END;
$$;

CREATE TRIGGER trg_media_entry_stats
    AFTER UPDATE OF status ON user_media_entries
    FOR EACH ROW EXECUTE FUNCTION fn_update_stats_on_media_entry();

-- Trigger: follower/following count changes
CREATE OR REPLACE FUNCTION fn_update_follow_counts()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        UPDATE user_stats
            SET following_count = following_count + 1, updated_at = NOW()
            WHERE user_id = NEW.follower_id;
        UPDATE user_stats
            SET follower_count = follower_count + 1, updated_at = NOW()
            WHERE user_id = NEW.following_id;
    ELSIF TG_OP = 'DELETE' THEN
        UPDATE user_stats
            SET following_count = GREATEST(following_count - 1, 0), updated_at = NOW()
            WHERE user_id = OLD.follower_id;
        UPDATE user_stats
            SET follower_count = GREATEST(follower_count - 1, 0), updated_at = NOW()
            WHERE user_id = OLD.following_id;
    END IF;
    RETURN NULL;
END;
$$;

CREATE TRIGGER trg_follows_stats
    AFTER INSERT OR DELETE ON follows
    FOR EACH ROW EXECUTE FUNCTION fn_update_follow_counts();

-- Trigger: review count changes
CREATE OR REPLACE FUNCTION fn_update_review_count()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    IF TG_OP = 'INSERT' AND NEW.deleted_at IS NULL THEN
        UPDATE user_stats SET review_count = review_count + 1, updated_at = NOW()
            WHERE user_id = NEW.user_id;
    ELSIF TG_OP = 'UPDATE' THEN
        IF OLD.deleted_at IS NULL AND NEW.deleted_at IS NOT NULL THEN
            UPDATE user_stats
                SET review_count = GREATEST(review_count - 1, 0), updated_at = NOW()
                WHERE user_id = NEW.user_id;
        ELSIF OLD.deleted_at IS NOT NULL AND NEW.deleted_at IS NULL THEN
            UPDATE user_stats SET review_count = review_count + 1, updated_at = NOW()
                WHERE user_id = NEW.user_id;
        END IF;
    END IF;
    RETURN NULL;
END;
$$;

CREATE TRIGGER trg_reviews_stats
    AFTER INSERT OR UPDATE OF deleted_at ON reviews
    FOR EACH ROW EXECUTE FUNCTION fn_update_review_count();
```

---

## 18. Media Screenshots

```sql
-- ============================================================
-- TABLE: media_screenshots
-- Additional images for media (screenshots, alternate posters,
-- backdrops). Polymorphic on media_type + media_id.
-- ============================================================

CREATE TABLE media_screenshots (
    id                  UUID                NOT NULL DEFAULT gen_random_uuid(),
    media_type          media_type_enum     NOT NULL,
    media_id            UUID                NOT NULL,
    image_url           TEXT                NOT NULL,
    type                screenshot_type_enum NOT NULL DEFAULT 'screenshot',
    width               SMALLINT            NULL CHECK (width > 0),
    height              SMALLINT            NULL CHECK (height > 0),
    language            CHAR(5)             NULL,   -- BCP-47, NULL means no language (e.g., logo)
    vote_average        DECIMAL(4,2)        NULL CHECK (vote_average BETWEEN 0 AND 10),
    created_at          TIMESTAMPTZ         NOT NULL DEFAULT NOW(),

    CONSTRAINT pk_media_screenshots PRIMARY KEY (id)
);

CREATE INDEX idx_media_screenshots_media ON media_screenshots (media_type, media_id, type);
CREATE INDEX idx_media_screenshots_type ON media_screenshots (type);
```

---

## 19. Admin & Moderation

```sql
-- ============================================================
-- TABLE: admin_actions
-- Immutable audit log of all moderation actions taken by
-- admin users. Records are never soft-deleted or updated.
-- ============================================================

CREATE TABLE admin_actions (
    id                  UUID                    NOT NULL DEFAULT gen_random_uuid(),
    admin_id            UUID                    NOT NULL,   -- references users.id
    action_type         admin_action_type_enum  NOT NULL,
    target_type         TEXT                    NOT NULL,   -- 'review' | 'comment' | 'user' | 'list'
    target_id           UUID                    NOT NULL,
    reason              TEXT                    NOT NULL,
    metadata            JSONB                   NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ             NOT NULL DEFAULT NOW(),

    CONSTRAINT pk_admin_actions PRIMARY KEY (id),
    CONSTRAINT fk_admin_actions_admin FOREIGN KEY (admin_id)
        REFERENCES users (id) ON DELETE RESTRICT,   -- never delete admins with existing actions
    CONSTRAINT chk_admin_actions_reason CHECK (char_length(reason) >= 5)
);

CREATE INDEX idx_admin_actions_admin_id ON admin_actions (admin_id);
CREATE INDEX idx_admin_actions_target ON admin_actions (target_type, target_id);
CREATE INDEX idx_admin_actions_created_at ON admin_actions (created_at DESC);
CREATE INDEX idx_admin_actions_action_type ON admin_actions (action_type, created_at DESC);
```

---

## 20. Full-Text Search Configuration

```sql
-- ============================================================
-- FULL-TEXT SEARCH
-- Arabic text requires the 'arabic' text search configuration.
-- English content uses 'english' (with stemming & stop words).
-- A unified cross-language search function combines both.
-- ============================================================

-- Verify Arabic FTS config exists (requires the appropriate
-- PostgreSQL text search dictionary package)
-- CREATE TEXT SEARCH CONFIGURATION arabic (COPY = pg_catalog.simple);

-- Cross-language anime search function
CREATE OR REPLACE FUNCTION fn_anime_fts_query(query_text TEXT)
RETURNS TABLE (
    id          UUID,
    title_en    TEXT,
    title_romaji TEXT,
    rank        REAL
)
LANGUAGE sql STABLE AS $$
    SELECT
        a.id,
        a.title_en,
        a.title_romaji,
        ts_rank_cd(a.search_vector,
            websearch_to_tsquery('english', query_text) ||
            websearch_to_tsquery('simple', query_text)
        ) AS rank
    FROM anime a
    WHERE a.search_vector @@
        (websearch_to_tsquery('english', query_text) ||
         websearch_to_tsquery('simple', query_text))
    ORDER BY rank DESC;
$$;

-- Trigram similarity search for fuzzy matching on anime titles
CREATE OR REPLACE FUNCTION fn_anime_fuzzy_search(query_text TEXT, min_similarity REAL DEFAULT 0.3)
RETURNS TABLE (
    id          UUID,
    title_en    TEXT,
    similarity  REAL
)
LANGUAGE sql STABLE AS $$
    SELECT
        a.id,
        a.title_en,
        GREATEST(
            similarity(a.title_en, query_text),
            similarity(a.title_romaji, query_text)
        ) AS similarity
    FROM anime a
    WHERE
        a.title_en % query_text OR
        a.title_romaji % query_text
    ORDER BY similarity DESC
    LIMIT 20;
$$;
```

---

## 21. Row-Level Security Policies

```sql
-- ============================================================
-- ROW-LEVEL SECURITY
-- Enabled on user-facing tables. Application connects with a
-- role that has current_setting('app.current_user_id') set
-- at the start of each request via SET LOCAL.
-- ============================================================

ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE user_media_entries ENABLE ROW LEVEL SECURITY;
ALTER TABLE reviews ENABLE ROW LEVEL SECURITY;
ALTER TABLE lists ENABLE ROW LEVEL SECURITY;
ALTER TABLE activities ENABLE ROW LEVEL SECURITY;
ALTER TABLE favorites ENABLE ROW LEVEL SECURITY;
ALTER TABLE user_episode_progress ENABLE ROW LEVEL SECURITY;

-- Users: anyone can read public profiles; only self can read private
CREATE POLICY rls_users_select ON users
    FOR SELECT
    USING (
        deleted_at IS NULL AND (
            privacy_setting = 'public' OR
            id::TEXT = current_setting('app.current_user_id', TRUE) OR
            EXISTS (
                SELECT 1 FROM follows
                WHERE follower_id::TEXT = current_setting('app.current_user_id', TRUE)
                  AND following_id = users.id
            )
        )
    );

CREATE POLICY rls_users_update_self ON users
    FOR UPDATE
    USING (id::TEXT = current_setting('app.current_user_id', TRUE));

-- User media entries: respect user privacy_setting
CREATE POLICY rls_ume_select ON user_media_entries
    FOR SELECT
    USING (
        user_id::TEXT = current_setting('app.current_user_id', TRUE) OR
        EXISTS (
            SELECT 1 FROM users u
            WHERE u.id = user_id
              AND u.deleted_at IS NULL
              AND (
                  u.privacy_setting = 'public' OR
                  (u.privacy_setting = 'followers_only' AND EXISTS (
                      SELECT 1 FROM follows
                      WHERE follower_id::TEXT = current_setting('app.current_user_id', TRUE)
                        AND following_id = u.id
                  ))
              )
        )
    );

CREATE POLICY rls_ume_write_self ON user_media_entries
    FOR ALL
    USING (user_id::TEXT = current_setting('app.current_user_id', TRUE));

-- Reviews: only non-deleted reviews from non-deleted users are visible
CREATE POLICY rls_reviews_select ON reviews
    FOR SELECT
    USING (
        deleted_at IS NULL AND
        EXISTS (
            SELECT 1 FROM users u
            WHERE u.id = reviews.user_id AND u.deleted_at IS NULL
        )
    );

CREATE POLICY rls_reviews_write_self ON reviews
    FOR ALL
    USING (user_id::TEXT = current_setting('app.current_user_id', TRUE));

-- Lists: respect visibility setting
CREATE POLICY rls_lists_select ON lists
    FOR SELECT
    USING (
        user_id::TEXT = current_setting('app.current_user_id', TRUE) OR
        visibility = 'public' OR
        (visibility = 'followers_only' AND EXISTS (
            SELECT 1 FROM follows
            WHERE follower_id::TEXT = current_setting('app.current_user_id', TRUE)
              AND following_id = lists.user_id
        ))
    );

CREATE POLICY rls_lists_write_self ON lists
    FOR ALL
    USING (user_id::TEXT = current_setting('app.current_user_id', TRUE));

-- Activities: only see your own or followed users' activities
CREATE POLICY rls_activities_select ON activities
    FOR SELECT
    USING (
        user_id::TEXT = current_setting('app.current_user_id', TRUE) OR
        EXISTS (
            SELECT 1 FROM follows
            WHERE follower_id::TEXT = current_setting('app.current_user_id', TRUE)
              AND following_id = activities.user_id
        )
    );
```

---

## 22. Trigger Definitions

All trigger definitions are included in their respective table sections above. Summary of all triggers:

| Trigger | Table | Event | Action |
|---|---|---|---|
| `trg_users_updated_at` | `users` | BEFORE UPDATE | Set `updated_at = NOW()` |
| `trg_users_create_preferences` | `users` | AFTER INSERT | Create `user_preferences` row |
| `trg_users_create_stats` | `users` | AFTER INSERT | Create `user_stats` row |
| `trg_user_preferences_updated_at` | `user_preferences` | BEFORE UPDATE | Set `updated_at` |
| `trg_user_oauth_connections_updated_at` | `user_oauth_connections` | BEFORE UPDATE | Set `updated_at` |
| `trg_anime_updated_at` | `anime` | BEFORE UPDATE | Set `updated_at` |
| `trg_tv_series_updated_at` | `tv_series` | BEFORE UPDATE | Set `updated_at` |
| `trg_movies_updated_at` | `movies` | BEFORE UPDATE | Set `updated_at` |
| `trg_seasons_updated_at` | `seasons` | BEFORE UPDATE | Set `updated_at` |
| `trg_episodes_updated_at` | `episodes` | BEFORE UPDATE | Set `updated_at` |
| `trg_anime_episodes_updated_at` | `anime_episodes` | BEFORE UPDATE | Set `updated_at` |
| `trg_people_updated_at` | `people` | BEFORE UPDATE | Set `updated_at` |
| `trg_studios_updated_at` | `studios` | BEFORE UPDATE | Set `updated_at` |
| `trg_sync_jobs_updated_at` | `sync_jobs` | BEFORE UPDATE | Set `updated_at` |
| `trg_user_media_entries_updated_at` | `user_media_entries` | BEFORE UPDATE | Set `updated_at` |
| `trg_episode_watch_stats` | `user_episode_progress` | AFTER INSERT | Update `user_stats` episode count + watch time |
| `trg_media_entry_stats` | `user_media_entries` | AFTER UPDATE status | Update `user_stats` completion counts |
| `trg_follows_stats` | `follows` | AFTER INSERT/DELETE | Update `user_stats` follower/following counts |
| `trg_reviews_stats` | `reviews` | AFTER INSERT/UPDATE deleted_at | Update `user_stats` review count |
| `trg_review_likes_count` | `review_likes` | AFTER INSERT/DELETE | Update `reviews.like_count` |
| `trg_comments_count` | `comments` | AFTER INSERT/UPDATE deleted_at | Update `reviews.comment_count` |
| `trg_list_items_count` | `list_items` | AFTER INSERT/DELETE | Update `lists.item_count` |
| `trg_reviews_updated_at` | `reviews` | BEFORE UPDATE | Set `updated_at` |
| `trg_lists_updated_at` | `lists` | BEFORE UPDATE | Set `updated_at` |
| `trg_characters_updated_at` | `characters` | BEFORE UPDATE | Set `updated_at` |

---

## 23. Partitioning Strategy

```sql
-- ============================================================
-- PARTITIONING STRATEGY
-- Both `notifications` and `activities` are partitioned by
-- created_at using RANGE partitioning per calendar month.
-- pg_partman creates and maintains partitions automatically.
-- ============================================================

-- Install pg_partman extension (done once by DBA)
-- CREATE EXTENSION IF NOT EXISTS pg_partman SCHEMA partman;

-- Register notifications with pg_partman (run once)
-- SELECT partman.create_parent(
--     p_parent_table   => 'public.notifications',
--     p_control        => 'created_at',
--     p_type           => 'native',
--     p_interval       => 'monthly',
--     p_premake        => 3   -- create 3 future partitions in advance
-- );

-- Register activities with pg_partman
-- SELECT partman.create_parent(
--     p_parent_table   => 'public.activities',
--     p_control        => 'created_at',
--     p_type           => 'native',
--     p_interval       => 'monthly',
--     p_premake        => 3
-- );

-- Partition maintenance job (pg_cron, runs daily at 01:00 UTC)
-- SELECT cron.schedule('partition-maintenance', '0 1 * * *',
--     $$CALL partman.run_maintenance_proc()$$);

-- ============================================================
-- DATA RETENTION AUTOMATION
-- Old partitions are detached and dropped on schedule:
--   notifications: retain 90 days
--   activities:    retain 30 days
-- ============================================================

-- pg_partman retention config for notifications
-- UPDATE partman.part_config
--     SET retention = '90 days', retention_keep_table = FALSE
--     WHERE parent_table = 'public.notifications';

-- pg_partman retention config for activities
-- UPDATE partman.part_config
--     SET retention = '30 days', retention_keep_table = FALSE
--     WHERE parent_table = 'public.activities';

-- NOTES on partition design:
-- 1. Queries on notifications/activities MUST always include
--    a created_at predicate to enable partition pruning.
-- 2. The composite primary key (id, created_at) is required
--    because PostgreSQL range-partitioned tables must include
--    the partition key in the primary key.
-- 3. Foreign keys pointing INTO partitioned tables are not
--    supported by PostgreSQL; therefore notifications and
--    activities do not have inbound FKs from other tables.
-- 4. Global unique constraints across all partitions are not
--    possible; uniqueness on `id` is enforced at application
--    layer using UUID v4 collision probability guarantees.
```

---

## 24. Index Strategy Summary

### B-Tree Indexes (Equality / Range Queries)
- All `created_at` and `updated_at` columns on user-facing tables for chronological pagination.
- All `tmdb_id` and `anilist_id` columns for upsert operations during sync.
- `user_id` on every user-data table for per-user data access.
- `status` on `user_media_entries` for list filtering (watching, completed, planned, etc.).
- `popularity DESC` and `vote_average DESC` on media tables for trending/top-rated queries.
- `position ASC` on `list_items` and `favorites` for ordered retrieval.

### GIN Indexes (Full-Text & Array Queries)
- `search_vector` on `anime`, `tv_series`, `movies` for PostgreSQL FTS.
- `data` JSONB on `notifications` and `activities` for payload key lookups.
- `origin_country` TEXT[] on `tv_series` and `movies` for country filtering.
- `language_filter` TEXT[] on `user_preferences` (not indexed — small table scanned in full).

### GIN Trigram Indexes (Fuzzy Search)
- `title_en gin_trgm_ops` on `anime`, `tv_series`, `movies` for similarity-based title lookup.
- `name gin_trgm_ops` on `people` and `characters` for name search.

### Partial Indexes (Filtered Indexes for Common Patterns)
- `WHERE deleted_at IS NULL` on `users`, `reviews`, `comments` to exclude soft-deleted rows.
- `WHERE is_admin = TRUE` on `users` for admin permission checks.
- `WHERE read_at IS NULL` on `notifications` for unread-count queries.
- `WHERE status IN ('pending', 'completed', 'partial')` on `sync_jobs` for scheduler queue.
- `WHERE anilist_episode_id IS NOT NULL` on `anime_episodes` for external ID lookups.

### Composite Indexes (Multi-Column Access Patterns)
- `(user_id, media_type, media_id)` on `user_media_entries` for single-entry lookup.
- `(user_id, status)` on `user_media_entries` for "show my watching list" queries.
- `(media_type, media_id)` on all polymorphic tables for media detail page assembly.
- `(season_id, episode_number)` on `episodes` for episode list retrieval.
- `(user_id, created_at DESC)` on `activities` and `notifications` for feed pagination.
- `(follower_id)` and `(following_id)` on `follows` for social graph traversal.

---

## 25. Migration Strategy

### Tooling
- **golang-migrate** is used for versioned, sequential migrations.
- All migration files follow the naming convention: `NNNN_description_direction.sql` (e.g., `0001_create_users_up.sql`, `0001_create_users_down.sql`).
- Migrations are applied atomically within a transaction. If any statement fails, the entire migration rolls back.

### Zero-Downtime Migration Principles

1. **Additive-first**: New columns are added as `NULL` or with a `DEFAULT` before any NOT NULL constraint is added. This avoids table rewrites that lock the table.
2. **Index creation**: All new indexes are created with `CREATE INDEX CONCURRENTLY` to avoid blocking reads/writes during build.
3. **Column renames**: Performed in three steps — (1) add new column, (2) dual-write to old+new in application, (3) backfill, (4) migrate reads, (5) drop old column.
4. **Enum additions**: New enum values are added with `ALTER TYPE ... ADD VALUE` which is safe and does not require a table rewrite.
5. **Foreign key additions**: Added with `NOT VALID` first, then `VALIDATE CONSTRAINT` in a separate migration to avoid full table scans holding a lock.
6. **Large table backfills**: Executed in batches of 10,000 rows using a cursor loop in a migration function, committing per batch to avoid long-held locks and WAL bloat.

### Migration Order
```
0001_extensions_and_enums
0002_users_and_preferences
0003_oauth_connections
0004_follows
0005_anime
0006_tv_series
0007_movies
0008_seasons_and_episodes
0009_anime_episodes
0010_people_and_characters
0011_media_cast
0012_studios_and_networks
0013_genres_and_media_genres
0014_user_media_entries
0015_user_episode_progress
0016_reviews_and_likes
0017_comments
0018_lists_and_favorites
0019_notifications_partitioned
0020_activities_partitioned
0021_sync_jobs_and_conflicts
0022_user_stats_and_triggers
0023_media_screenshots
0024_admin_actions
0025_rls_policies
0026_fts_indexes
0027_partman_setup
```

---

## 26. Data Retention Policies

| Table | Retention Period | Mechanism |
|---|---|---|
| `activities` | 30 days | pg_partman drops old monthly partitions |
| `notifications` | 90 days | pg_partman drops old monthly partitions |
| `sync_conflicts` (resolved) | 180 days | pg_cron job deletes `WHERE resolved_at < NOW() - INTERVAL '180 days'` |
| `admin_actions` | Indefinite | Never deleted — regulatory compliance audit trail |
| `users` (soft-deleted) | 30 days post soft-delete | pg_cron job hard-deletes `WHERE deleted_at < NOW() - INTERVAL '30 days'` |
| `reviews` (soft-deleted) | 14 days post soft-delete | pg_cron job hard-deletes |
| `comments` (soft-deleted) | 14 days post soft-delete | pg_cron job hard-deletes |

```sql
-- Scheduled cleanup jobs (pg_cron)

-- Hard-delete soft-deleted users after 30 days grace period
-- SELECT cron.schedule('cleanup-deleted-users', '0 3 * * *', $$
--     DELETE FROM users
--     WHERE deleted_at IS NOT NULL
--       AND deleted_at < NOW() - INTERVAL '30 days';
-- $$);

-- Hard-delete soft-deleted reviews after 14 days
-- SELECT cron.schedule('cleanup-deleted-reviews', '0 3 * * *', $$
--     DELETE FROM reviews
--     WHERE deleted_at IS NOT NULL
--       AND deleted_at < NOW() - INTERVAL '14 days';
-- $$);

-- Clean up old resolved sync conflicts
-- SELECT cron.schedule('cleanup-sync-conflicts', '0 4 * * 0', $$
--     DELETE FROM sync_conflicts
--     WHERE resolved_at IS NOT NULL
--       AND resolved_at < NOW() - INTERVAL '180 days';
-- $$);
```

---

## 27. Soft Delete Pattern

All user-generated content and user identity records implement soft deletes via a `deleted_at TIMESTAMPTZ NULL` column.

### Rules

1. **Application layer**: All queries that fetch user-visible data append `WHERE deleted_at IS NULL` to exclude deleted records. This is enforced via a query builder base scope and is tested in integration tests.

2. **Partial unique indexes**: The `uq_reviews_active_user_media` partial index covers only `WHERE deleted_at IS NULL`, allowing a user to delete their review and then re-review the same title later without violating the uniqueness constraint.

3. **Cascade behavior**: When a `users` row is soft-deleted (setting `deleted_at`), child records are NOT automatically soft-deleted. Instead:
   - The user's profile is hidden via the `rls_users_select` RLS policy.
   - The user's content (reviews, comments) is hidden via join conditions on `users.deleted_at IS NULL`.
   - After the 30-day grace period, the hard DELETE on `users` cascades to all child records via `ON DELETE CASCADE` foreign keys.

4. **Admin visibility**: Admin queries bypass RLS (connecting as the `octotime_admin` role) and can view soft-deleted records explicitly by filtering `WHERE deleted_at IS NOT NULL`.

5. **Audit trail**: `admin_actions` records are never soft-deleted and do not have a `deleted_at` column. They are append-only.

6. **GDPR compliance**: The hard delete after the grace period ensures right-to-erasure compliance. The `email` column is anonymized at soft-delete time (replaced with `deleted_<uuid>@deleted.octotime.com`) to free up the unique constraint immediately.

```sql
-- GDPR soft-delete procedure
CREATE OR REPLACE PROCEDURE proc_soft_delete_user(p_user_id UUID)
LANGUAGE plpgsql AS $$
BEGIN
    UPDATE users SET
        deleted_at   = NOW(),
        email        = 'deleted_' || p_user_id::TEXT || '@deleted.octotime.com',
        password_hash = NULL,
        avatar_url   = NULL,
        banner_url   = NULL,
        display_name = 'Deleted User'
    WHERE id = p_user_id AND deleted_at IS NULL;

    -- Revoke all OAuth connections immediately
    DELETE FROM user_oauth_connections WHERE user_id = p_user_id;

    -- Invalidate sessions (done in application layer via Redis)
END;
$$;
```
