# Octo Time

A cross-platform media tracking platform for Anime, TV Series, and Movies.

## Overview

Octo Time is a premium tracking and social experience — not a streaming platform. Users track their watching history, discover new content, write reviews, and connect with other fans.

### Supported Media
- **Anime** — via AniList (import/export/sync)
- **TV Series** — via TMDB
- **Movies** — via TMDB

### External Sync
- AniList
- MyAnimeList
- Trakt
- Letterboxd

### Platforms
1. Web Application (Desktop + Mobile Web) — primary
2. iPhone Application
3. Android Application

## Documentation

All design, architecture, and product documentation lives in [`/docs`](./docs/).

| # | Document |
|---|----------|
| 01 | [Product Specification](./docs/01-product-specification.md) |
| 02 | [User Flows](./docs/02-user-flows.md) |
| 03 | [Information Architecture](./docs/03-information-architecture.md) |
| 04 | [Database Schema](./docs/04-database-schema.md) |
| 05 | [Entity Relationship Diagram](./docs/05-erd.md) |
| 06 | [API Design](./docs/06-api-design.md) |
| 07 | [Backend Architecture](./docs/07-backend-architecture.md) |
| 08 | [Frontend Architecture](./docs/08-frontend-architecture.md) |
| 09 | [Mobile Architecture](./docs/09-mobile-architecture.md) |
| 10 | [Authentication Flow](./docs/10-authentication-flow.md) |
| 11 | [Synchronization Architecture](./docs/11-synchronization-architecture.md) |
| 12 | [Notification Architecture](./docs/12-notification-architecture.md) |
| 13 | [Search Architecture](./docs/13-search-architecture.md) |
| 14 | [Statistics Engine Design](./docs/14-statistics-engine.md) |
| 15 | [UI/UX Design System](./docs/15-ui-ux-design-system.md) |
| 16 | [Design Tokens](./docs/16-design-tokens.md) |
| 17 | [Component Library](./docs/17-component-library.md) |
| 18 | [Development Roadmap](./docs/18-development-roadmap.md) |
| 19 | [MVP Breakdown](./docs/19-mvp-breakdown.md) |
| 20 | [Post-MVP Roadmap](./docs/20-post-mvp-roadmap.md) |
| 21 | [Deployment Architecture](./docs/21-deployment-architecture.md) |
| 22 | [Scaling Strategy](./docs/22-scaling-strategy.md) |
| 23 | [Security Requirements](./docs/23-security-requirements.md) |
| 24 | [Testing Strategy](./docs/24-testing-strategy.md) |
| 25 | [Folder Structure](./docs/25-folder-structure.md) |
| 26 | [Technology Stack](./docs/26-technology-stack.md) |

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Backend | Node.js 22 + TypeScript + Fastify 5 |
| Database | PostgreSQL 16 + Redis 7 + Elasticsearch 8 |
| Queue | BullMQ |
| Web Frontend | Next.js 15 (App Router) + Tailwind CSS v4 |
| Mobile | React Native + Expo SDK 52 |
| Monorepo | pnpm workspaces + Turborepo |
| Infrastructure | Kubernetes (EKS) + Terraform + Cloudflare |

## Languages
- English (LTR)
- Arabic (RTL)
