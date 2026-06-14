# 08 — Frontend (Web) Architecture

## Overview

The Octo Time web frontend is a **Next.js 15 App Router application** with full TypeScript, supporting Arabic and English locales (with RTL layout), three colour themes (light/dark/OLED), and a performance-first rendering strategy that mixes React Server Components for data-heavy pages with Client Components only where interactivity demands it.

---

## Technology Stack

| Concern | Choice | Notes |
|---|---|---|
| Framework | Next.js 15 (App Router) | React 19, Server Components, Streaming, PPR-ready |
| Language | TypeScript 5.x (strict) | `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes` enabled |
| Styling | Tailwind CSS v4 + CSS custom properties | No Tailwind config file; CSS-first config in `app.css` |
| Global state | Zustand 5 | Auth, theme, UI state |
| Server state | TanStack Query v5 | Caching, background refetch, optimistic updates |
| Animation | Framer Motion 12 | Page transitions, list animations, micro-interactions |
| Component base | Custom component library | shadcn/ui-inspired patterns; components owned and versioned in-repo |
| Icons | Lucide React | Tree-shakeable, consistent stroke-width |
| Forms | React Hook Form v7 + Zod | Schema-driven validation matching backend Zod schemas |
| i18n | next-intl v3 | Server + client components, ICU messages, RTL routing |
| Date handling | date-fns v3 | Locale-aware formatting |
| HTTP client | ky (fetch wrapper) | Interceptors for auth headers, token refresh |
| Testing | Vitest + React Testing Library | Unit and component tests |
| E2E | Playwright | Full browser tests against local dev server |

---

## Architecture Patterns

### Server vs Client Components

The default is **Server Components**. A component opts into the client only when it needs:
- Event handlers (`onClick`, `onChange`, etc.)
- Browser APIs (`localStorage`, `window`, etc.)
- React state or effects
- TanStack Query hooks
- Framer Motion (requires client context)

Server Components fetch data directly from the backend using the internal API client (server-side `ky` instance with service token). They never expose API keys or internal tokens to the browser.

Client Components receive serializable props from their Server Component parents and call the API through route handlers or directly with the user's session token.

### Feature-Based Folder Structure

All code belonging to a feature lives together. `app/` handles routing; `features/` holds all business logic, components, hooks, and types.

### BFF (Backend for Frontend) via Route Handlers

Some requests require server-side processing before reaching the browser:
- OAuth callback handling (never expose access tokens to the client)
- Combining multiple API calls into one response (fan-in)
- Rewriting/filtering responses for mobile viewport

These are implemented as Next.js Route Handlers in `app/api/`.

### Middleware

Next.js middleware (`middleware.ts`) handles:
1. **Auth check:** Redirects unauthenticated users from protected routes to `/login`
2. **i18n routing:** Detects locale from cookie/Accept-Language header, redirects to localised path (`/ar/*`, `/en/*`)
3. **Bot detection:** Blocks headless scrapers on user-profile routes (rate-limit header injection)

---

## Routing

### Route Map

```
/                              → Home (discovery, trending, seasonal)
/[locale]/                     → Localised home
/login                         → Auth page (login + register tabs)
/u/[username]                  → User profile
/u/[username]/anime            → User's anime list
/u/[username]/tv               → User's TV list
/u/[username]/movies           → User's movie list
/u/[username]/reviews          → User's reviews
/u/[username]/lists            → User's custom lists
/u/[username]/activity         → Activity feed
/u/[username]/stats            → Statistics page
/anime                         → Anime tracking hub (auth required)
/tv                            → TV tracking hub (auth required)
/movies                        → Movie tracking hub (auth required)
/anime/[slug]                  → Anime detail page
/tv/[slug]                     → TV series detail page
/movie/[slug]                  → Movie detail page
/list/[slug]                   → Public list view
/search                        → Search page
/notifications                 → Notifications (auth required)
/settings                      → Settings hub (auth required)
/settings/account              → Account settings
/settings/profile              → Profile settings
/settings/privacy              → Privacy settings
/settings/notifications        → Notification preferences
/settings/connections          → External service connections (sync)
/settings/appearance           → Theme, language settings
/settings/data                 → Export, delete account
/characters/[id]               → Character detail page
/people/[id]                   → Staff / person detail page
/staff/[id]                    → Alias for /people/[id]
```

### Route Segment Config

Media detail pages use ISR:
```typescript
// app/anime/[slug]/page.tsx
export const revalidate = 3600 // 1 hour
export const dynamicParams = true

export async function generateStaticParams() {
  // Pre-render top 500 anime by popularity
  return getTopAnime(500).then(list => list.map(a => ({ slug: a.slug })))
}
```

---

## State Management

### Zustand Stores

```typescript
// Auth store — persisted to cookie (server-readable for SSR)
interface AuthStore {
  user: UserProfile | null
  isAuthenticated: boolean
  login: (user: UserProfile, tokens: Tokens) => void
  logout: () => void
  updateUser: (partial: Partial<UserProfile>) => void
}

// Theme store — persisted to localStorage + synced to DB on change
interface ThemeStore {
  theme: 'light' | 'dark' | 'oled' | 'system'
  setTheme: (theme: Theme) => void
}

// UI store — ephemeral, not persisted
interface UIStore {
  sidebarOpen: boolean
  activeModal: ModalType | null
  openModal: (type: ModalType, props?: unknown) => void
  closeModal: () => void
}

// Tracking optimistic store — local-first episode marking
interface TrackingOptimisticStore {
  pendingEpisodes: Set<string>           // episodeId strings being saved
  markPending: (episodeId: string) => void
  markResolved: (episodeId: string) => void
}
```

### TanStack Query Configuration

```typescript
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5,          // 5 minutes — matches Redis profile cache TTL
      gcTime: 1000 * 60 * 30,            // 30 minutes in memory
      retry: (failureCount, error) => {
        if (error instanceof ApiError && error.status < 500) return false
        return failureCount < 3
      },
      refetchOnWindowFocus: false,
    },
    mutations: {
      onError: (error) => toast.error(getErrorMessage(error)),
    },
  },
})
```

### Optimistic Updates

Episode marking uses an optimistic update pattern — the UI flips the checkbox immediately, the mutation runs in background, and on error the optimistic state is rolled back:

```typescript
const { mutate: markWatched } = useMutation({
  mutationFn: (episodeId: string) => api.tracking.markEpisodeWatched(mediaId, episodeId),
  onMutate: async (episodeId) => {
    await queryClient.cancelQueries({ queryKey: ['progress', mediaId] })
    const previous = queryClient.getQueryData(['progress', mediaId])
    queryClient.setQueryData(['progress', mediaId], (old) => ({
      ...old,
      watchedEpisodes: [...old.watchedEpisodes, episodeId],
    }))
    return { previous }
  },
  onError: (err, episodeId, ctx) => {
    queryClient.setQueryData(['progress', mediaId], ctx.previous)
  },
  onSettled: () => {
    queryClient.invalidateQueries({ queryKey: ['progress', mediaId] })
    queryClient.invalidateQueries({ queryKey: ['stats'] })
  },
})
```

---

## Theme System

### CSS Custom Properties

All design tokens are CSS custom properties on the `:root` and `[data-theme]` selectors. Tailwind v4 reads these properties directly — no `tailwind.config.js` colour definitions.

```css
/* app/styles/tokens.css */
:root {
  /* Applied for [data-theme="light"] */
  --color-bg:           #ffffff;
  --color-bg-subtle:    #f4f4f5;
  --color-bg-elevated:  #ffffff;
  --color-border:       #e4e4e7;
  --color-text:         #09090b;
  --color-text-muted:   #71717a;
  --color-primary:      #6366f1;   /* Indigo — Octo brand */
  --color-primary-fg:   #ffffff;
  --color-accent:       #f472b6;
  --color-success:      #22c55e;
  --color-warning:      #f59e0b;
  --color-danger:       #ef4444;

  /* Spacing scale */
  --space-1: 0.25rem;
  --space-2: 0.5rem;
  /* ... */

  /* Typography */
  --font-sans: 'Inter Variable', system-ui, sans-serif;
  --font-sans-ar: 'Cairo Variable', 'Tajawal', sans-serif;
  --radius-md: 0.5rem;
  --radius-lg: 0.75rem;
}

[data-theme="dark"] {
  --color-bg:           #09090b;
  --color-bg-subtle:    #18181b;
  --color-bg-elevated:  #27272a;
  --color-border:       #3f3f46;
  --color-text:         #fafafa;
  --color-text-muted:   #a1a1aa;
  /* primary and accent unchanged */
}

[data-theme="oled"] {
  --color-bg:           #000000;
  --color-bg-subtle:    #0a0a0a;
  --color-bg-elevated:  #111111;
  --color-border:       #27272a;
  --color-text:         #fafafa;
  --color-text-muted:   #71717a;
}
```

### Theme Application

The `data-theme` attribute is applied on the `<html>` element server-side (read from cookie) to prevent flash of wrong theme:

```typescript
// app/layout.tsx
export default function RootLayout({ children, params: { locale } }) {
  const theme = cookies().get('theme')?.value ?? 'dark'
  const dir = locale === 'ar' ? 'rtl' : 'ltr'

  return (
    <html lang={locale} dir={dir} data-theme={theme}>
      <body className={cn(fontInter.variable, fontCairo.variable)}>
        {children}
      </body>
    </html>
  )
}
```

---

## Internationalisation (i18n)

### Locale Support

| Locale | Language | Direction | Fonts |
|---|---|---|---|
| `en` | English | LTR | Inter Variable |
| `ar` | Arabic | RTL | Cairo Variable, Tajawal |

### next-intl Setup

```typescript
// i18n/request.ts
export default getRequestConfig(async ({ locale }) => ({
  messages: (await import(`../messages/${locale}.json`)).default,
  formats: {
    dateTime: {
      short: { day: 'numeric', month: 'short', year: 'numeric' }
    },
    number: {
      percent: { style: 'percent', minimumFractionDigits: 0 }
    }
  }
}))
```

Translation files are `messages/en.json` and `messages/ar.json`. All user-visible strings go through `useTranslations()` (client) or `getTranslations()` (server).

### RTL Support

CSS logical properties are used everywhere:
- `margin-inline-start` instead of `margin-left`
- `padding-inline-end` instead of `padding-right`
- `border-inline-start` instead of `border-left`
- `inset-inline-end` instead of `right`

Tailwind v4 exposes logical property utilities natively (`ms-4`, `pe-4`, `border-s`, `end-0`, etc.).

Framer Motion animations are direction-aware:
```typescript
const dir = useDirection() // from next-intl
const slideFrom = dir === 'rtl' ? 100 : -100
const variants = {
  hidden: { x: slideFrom, opacity: 0 },
  visible: { x: 0, opacity: 1 },
}
```

---

## Performance

### Rendering Strategy per Route

| Route | Strategy | Reason |
|---|---|---|
| `/` (Home) | SSR + Streaming | Personalised trending, session-dependent |
| `/anime/[slug]` | ISR (1hr) + Streaming | Mostly static media data |
| `/u/[username]` | SSR | Profile must be current |
| `/search` | CSR (client-side) | Query-driven, not indexable |
| `/[username]/anime` | SSR + streaming | List data must be fresh |
| `/settings/*` | CSR | Auth-gated, no SEO value |
| `/notifications` | CSR | Real-time, user-specific |

### Streaming with Suspense

Heavy page sections are wrapped in `<Suspense>` with skeleton fallbacks so the shell renders immediately:

```tsx
// app/anime/[slug]/page.tsx
export default function AnimeDetailPage({ params }) {
  return (
    <>
      <AnimeHero slug={params.slug} />          {/* Awaited at top level */}
      <Suspense fallback={<EpisodeListSkeleton />}>
        <EpisodeList slug={params.slug} />
      </Suspense>
      <Suspense fallback={<ReviewListSkeleton />}>
        <ReviewList slug={params.slug} />
      </Suspense>
      <Suspense fallback={<RelatedAnimeSkeleton />}>
        <RelatedAnime slug={params.slug} />
      </Suspense>
    </>
  )
}
```

### Image Optimisation

All images use `next/image` with:
- Cloudflare R2 CDN as the image source (configured in `next.config.ts` `images.remotePatterns`)
- `sizes` prop matching the actual responsive layout breakpoints
- `loading="lazy"` except above-the-fold hero images (`priority`)
- WebP/AVIF format negotiation handled by Next.js Image Optimization API (or Cloudflare Images)

### Prefetching

- `<Link>` components prefetch on hover/focus by default (Next.js behaviour)
- TanStack Query `prefetchQuery` on route transitions for known data needs
- `router.prefetch()` in `onMouseEnter` on media cards for detail pages

### Service Worker (Offline)

A minimal Workbox service worker provides:
- Cache-first for static assets (JS, CSS, fonts)
- Network-first for API responses with 5-second timeout fallback to cache
- Offline fallback page (`/offline`) for navigation requests

Registered via `next-pwa` or a custom `public/sw.js` (opt-in, not intrusive).

---

## Component Architecture

### Atomic Design Hierarchy

```
atoms/         → Button, Input, Badge, Avatar, Spinner, Skeleton, Icon
molecules/     → SearchInput, RatingStars, StatusBadge, EpisodeCheckbox, ThemeToggle
organisms/     → MediaCard, EpisodeList, ReviewCard, ProfileHeader, ListCard, NotificationItem
templates/     → PageLayout, ProfileLayout, MediaDetailLayout, SettingsLayout
pages/         → Assembled from templates + organisms (live in app/ routing)
```

### Compound Components

Complex UI patterns use the compound component pattern for flexible composition:

```tsx
// Usage
<MediaCard>
  <MediaCard.Poster src={anime.posterUrl} alt={anime.title} />
  <MediaCard.Body>
    <MediaCard.Title>{anime.title}</MediaCard.Title>
    <MediaCard.Meta>{anime.year} · {anime.episodes} eps</MediaCard.Meta>
    <MediaCard.Score value={anime.score} />
  </MediaCard.Body>
  <MediaCard.Actions>
    <TrackingButton mediaId={anime.id} mediaType="anime" />
  </MediaCard.Actions>
</MediaCard>
```

### Custom Hooks

```typescript
// Tracking
useTrackingEntry(mediaType, mediaId)       // current status + progress
useMarkEpisodeWatched(mediaId)             // mutation + optimistic update
useUpdateTrackingStatus(mediaId)           // status change mutation

// Auth
useCurrentUser()                           // current user from Zustand
useRequireAuth(redirectTo?)                // redirect if not authed

// Media
useAnimeDetail(slug)                       // TanStack Query wrapper
useEpisodeList(animeId, options?)

// Search
useSearch(query, filters)                  // debounced search hook
useSearchSuggestions(query)                // autocomplete

// UI
useDebounce(value, ms)
useMediaQuery(query)
useIntersectionObserver(ref, options)      // for infinite scroll
useDirection()                             // 'ltr' | 'rtl' from locale
```

---

## Key Page Designs

### Home Page (`/`)

```
Server Component tree:
  HomeLayout
  ├── HeroSection            (seasonal anime spotlight — ISR)
  ├── <Suspense>
  │     ContinueWatching     (auth'd users only — SSR, session-dependent)
  ├── <Suspense>
  │     TrendingRow          (anime / tv / movies tabs — ISR 15min)
  ├── <Suspense>
  │     NewEpisodesRow       (aired this week — ISR 1hr)
  └── <Suspense>
        RecommendedRow       (personalised — SSR if auth'd)
```

### Media Detail Page (`/anime/[slug]`)

```
AnimeDetailLayout
├── AnimeHero
│   ├── Poster (next/image, priority)
│   ├── Title (localised: English + Romaji + Native)
│   ├── MetaBadges (year, status, episodes, score)
│   ├── GenreTags
│   └── TrackingPanel (Client Component — status picker, score, progress)
├── SynopsisSection (expandable)
├── EpisodeList (Client Component — checkboxes for progress)
├── CharacterGrid
├── StaffGrid
├── ReviewList (with "Write a Review" CTA)
├── RelatedAnime
└── ExternalLinks (AniList, MAL, etc.)
```

### User Profile (`/u/[username]`)

```
ProfileLayout
├── ProfileHeader
│   ├── AvatarWithStatus
│   ├── DisplayName + Username
│   ├── Bio
│   ├── JoinDate, FollowerCount
│   └── FollowButton / EditProfileButton (conditional)
├── StatsBar (episodes, movies, time)
└── ProfileTabs (Client Component — tab state in URL)
    ├── /u/:username           → ActivityFeed
    ├── /u/:username/anime     → TrackingList (anime)
    ├── /u/:username/tv        → TrackingList (tv)
    ├── /u/:username/movies    → TrackingList (movies)
    ├── /u/:username/reviews   → ReviewGrid
    ├── /u/:username/lists     → ListGrid
    └── /u/:username/stats     → StatsCharts
```

---

## Forms

All forms use React Hook Form with Zod resolvers. The Zod schema is imported from the shared `packages/types` package (same schema used on the backend).

```typescript
// features/review/components/ReviewForm.tsx
'use client'
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { CreateReviewSchema, type CreateReviewInput } from '@octotime/types'

export function ReviewForm({ mediaId, mediaType, onSuccess }) {
  const { register, handleSubmit, formState: { errors, isSubmitting } } = useForm<CreateReviewInput>({
    resolver: zodResolver(CreateReviewSchema),
    defaultValues: { containsSpoilers: false },
  })

  const { mutate: createReview } = useCreateReview(mediaId, mediaType)

  return (
    <form onSubmit={handleSubmit((data) => createReview(data, { onSuccess }))}>
      <Textarea {...register('body')} label={t('review.bodyLabel')} error={errors.body?.message} />
      <RatingInput {...register('score')} label={t('review.scoreLabel')} />
      <Checkbox {...register('containsSpoilers')} label={t('review.spoilerWarning')} />
      <Button type="submit" loading={isSubmitting}>{t('review.submit')}</Button>
    </form>
  )
}
```

---

## Folder Structure

```
apps/web/
├── app/
│   ├── layout.tsx                          # Root layout (html, fonts, providers)
│   ├── page.tsx                            # Home (redirects to /[locale])
│   ├── [locale]/
│   │   ├── layout.tsx                      # Locale layout (i18n provider)
│   │   ├── page.tsx                        # Home page
│   │   ├── login/
│   │   │   └── page.tsx
│   │   ├── u/
│   │   │   └── [username]/
│   │   │       ├── page.tsx               # Profile overview (activity)
│   │   │       ├── anime/page.tsx
│   │   │       ├── tv/page.tsx
│   │   │       ├── movies/page.tsx
│   │   │       ├── reviews/page.tsx
│   │   │       ├── lists/page.tsx
│   │   │       └── stats/page.tsx
│   │   ├── anime/
│   │   │   ├── page.tsx                   # Tracking hub
│   │   │   └── [slug]/page.tsx            # Anime detail
│   │   ├── tv/
│   │   │   ├── page.tsx
│   │   │   └── [slug]/page.tsx
│   │   ├── movie/
│   │   │   ├── page.tsx
│   │   │   └── [slug]/page.tsx
│   │   ├── list/
│   │   │   └── [slug]/page.tsx
│   │   ├── search/page.tsx
│   │   ├── notifications/page.tsx
│   │   ├── characters/[id]/page.tsx
│   │   ├── people/[id]/page.tsx
│   │   └── settings/
│   │       ├── layout.tsx                 # Settings sidebar layout
│   │       ├── page.tsx                   # Redirect to /settings/profile
│   │       ├── account/page.tsx
│   │       ├── profile/page.tsx
│   │       ├── privacy/page.tsx
│   │       ├── notifications/page.tsx
│   │       ├── connections/page.tsx
│   │       ├── appearance/page.tsx
│   │       └── data/page.tsx
│   ├── api/
│   │   ├── auth/
│   │   │   ├── [...nextauth]/route.ts     # OAuth callbacks (BFF)
│   │   │   ├── refresh/route.ts
│   │   │   └── logout/route.ts
│   │   └── upload/route.ts               # Avatar upload (streams to R2)
│   ├── globals.css                        # Tailwind v4 @import, token definitions
│   └── error.tsx / not-found.tsx / loading.tsx
├── features/
│   ├── auth/
│   │   ├── components/
│   │   │   ├── LoginForm.tsx
│   │   │   ├── RegisterForm.tsx
│   │   │   └── OAuthButtons.tsx
│   │   ├── hooks/
│   │   │   ├── useLogin.ts
│   │   │   └── useCurrentUser.ts
│   │   └── store/
│   │       └── auth.store.ts
│   ├── tracking/
│   │   ├── components/
│   │   │   ├── TrackingPanel.tsx          # Status picker + score
│   │   │   ├── EpisodeList.tsx
│   │   │   ├── EpisodeCheckbox.tsx
│   │   │   └── TrackingListTable.tsx
│   │   ├── hooks/
│   │   │   ├── useTrackingEntry.ts
│   │   │   └── useMarkEpisodeWatched.ts
│   │   └── store/
│   │       └── tracking-optimistic.store.ts
│   ├── media/
│   │   ├── components/
│   │   │   ├── MediaCard.tsx
│   │   │   ├── AnimeHero.tsx
│   │   │   ├── MediaGrid.tsx
│   │   │   └── GenreTag.tsx
│   │   └── hooks/
│   │       ├── useAnimeDetail.ts
│   │       └── useTrending.ts
│   ├── review/
│   ├── list/
│   ├── search/
│   ├── notifications/
│   ├── user/
│   ├── discovery/
│   ├── stats/
│   └── settings/
├── components/
│   ├── atoms/
│   │   ├── Button.tsx
│   │   ├── Input.tsx
│   │   ├── Textarea.tsx
│   │   ├── Badge.tsx
│   │   ├── Avatar.tsx
│   │   ├── Spinner.tsx
│   │   ├── Skeleton.tsx
│   │   └── Icon.tsx
│   ├── molecules/
│   │   ├── SearchInput.tsx
│   │   ├── RatingStars.tsx
│   │   ├── StatusBadge.tsx
│   │   ├── ThemeToggle.tsx
│   │   ├── LanguageToggle.tsx
│   │   └── InfiniteScroll.tsx
│   └── organisms/
│       ├── Navbar.tsx
│       ├── Sidebar.tsx
│       ├── MediaCard.tsx
│       ├── ReviewCard.tsx
│       ├── ListCard.tsx
│       ├── NotificationItem.tsx
│       ├── ProfileHeader.tsx
│       └── Modal.tsx
├── lib/
│   ├── api/
│   │   ├── client.ts                      # ky instance with interceptors
│   │   ├── auth.api.ts
│   │   ├── tracking.api.ts
│   │   ├── media.api.ts
│   │   ├── review.api.ts
│   │   ├── search.api.ts
│   │   └── index.ts
│   ├── query/
│   │   └── client.ts                      # QueryClient singleton
│   ├── utils/
│   │   ├── cn.ts                          # clsx + twMerge utility
│   │   ├── format-date.ts
│   │   ├── format-duration.ts
│   │   └── slugify.ts
│   └── constants/
│       ├── tracking-statuses.ts
│       └── media-types.ts
├── middleware.ts                           # Auth + i18n middleware
├── messages/
│   ├── en.json
│   └── ar.json
├── public/
│   ├── fonts/
│   ├── icons/
│   └── sw.js
├── tests/
│   ├── unit/
│   ├── components/
│   └── e2e/
│       └── playwright/
├── next.config.ts
├── tailwind.css                            # v4 CSS entry (replaces config file)
├── vitest.config.ts
├── playwright.config.ts
├── tsconfig.json
└── package.json
```

---

## API Client

```typescript
// lib/api/client.ts
import ky, { type KyInstance } from 'ky'

let refreshPromise: Promise<void> | null = null

function createApiClient(baseUrl: string): KyInstance {
  return ky.create({
    prefixUrl: baseUrl,
    credentials: 'include',
    hooks: {
      beforeRequest: [
        (request) => {
          const token = getAccessToken()
          if (token) request.headers.set('Authorization', `Bearer ${token}`)
        },
      ],
      afterResponse: [
        async (request, options, response) => {
          if (response.status !== 401) return

          // Prevent concurrent refresh calls
          if (!refreshPromise) {
            refreshPromise = refreshAccessToken().finally(() => {
              refreshPromise = null
            })
          }
          await refreshPromise

          // Retry original request with new token
          const newToken = getAccessToken()
          if (newToken) request.headers.set('Authorization', `Bearer ${newToken}`)
          return ky(request)
        },
      ],
    },
  })
}

export const api = createApiClient(process.env.NEXT_PUBLIC_API_URL!)
```

---

## Environment Variables

```bash
# Public (exposed to browser)
NEXT_PUBLIC_API_URL=https://api.octotime.app
NEXT_PUBLIC_APP_URL=https://octotime.app
NEXT_PUBLIC_CDN_URL=https://media.octotime.app
NEXT_PUBLIC_SENTRY_DSN=...
NEXT_PUBLIC_POSTHOG_KEY=...      # Analytics (Phase 2)

# Server-only (not exposed to browser)
NEXTAUTH_SECRET=...              # For NextAuth if used
NEXTAUTH_URL=https://octotime.app
INTERNAL_API_TOKEN=...           # Service-to-service token for SSR API calls
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...
DISCORD_CLIENT_ID=...
DISCORD_CLIENT_SECRET=...
```

---

## Build Configuration

```typescript
// next.config.ts
import type { NextConfig } from 'next'
import createNextIntlPlugin from 'next-intl/plugin'

const withNextIntl = createNextIntlPlugin('./i18n/request.ts')

const config: NextConfig = {
  images: {
    remotePatterns: [
      { protocol: 'https', hostname: 'media.octotime.app' },
      { protocol: 'https', hostname: '**.anilist.co' },
      { protocol: 'https', hostname: 'image.tmdb.org' },
    ],
    formats: ['image/avif', 'image/webp'],
  },
  experimental: {
    ppr: true,                              // Partial Pre-Rendering (Next.js 15)
    reactCompiler: true,                    // React Compiler (beta)
    turbo: {},                              // Turbopack for dev
  },
  headers: async () => [
    {
      source: '/(.*)',
      headers: [
        { key: 'X-Frame-Options', value: 'DENY' },
        { key: 'X-Content-Type-Options', value: 'nosniff' },
        { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
      ],
    },
  ],
  // Redirect naked locale to preferred
  redirects: async () => [
    { source: '/', destination: '/en', permanent: false },
  ],
}

export default withNextIntl(config)
```

---

## Testing Strategy

### Unit Tests (Vitest)

- Pure utility functions in `lib/utils/`
- Zustand store actions
- API response transformers

### Component Tests (Vitest + React Testing Library)

- Render components with mock TanStack Query providers
- Test user interactions (click, type, submit)
- Test conditional rendering based on auth state
- Test i18n strings render correctly
- Accessibility assertions (ARIA roles, labels)

```typescript
// tests/components/TrackingPanel.test.tsx
it('marks episode as watched on checkbox click', async () => {
  const user = userEvent.setup()
  const mutate = vi.fn()
  vi.mocked(useMarkEpisodeWatched).mockReturnValue({ mutate })

  render(<EpisodeCheckbox episodeId="ep-1" watched={false} />, { wrapper: TestProviders })
  await user.click(screen.getByRole('checkbox'))
  expect(mutate).toHaveBeenCalledWith('ep-1')
})
```

### E2E Tests (Playwright)

```typescript
// tests/e2e/tracking.spec.ts
test('user can mark an episode as watched', async ({ page }) => {
  await loginAs(page, testUser)
  await page.goto('/anime/attack-on-titan')
  const checkbox = page.getByTestId('episode-checkbox-1')
  await checkbox.click()
  await expect(checkbox).toBeChecked()
  await expect(page.getByTestId('progress-bar')).toHaveAttribute('aria-valuenow', '1')
})
```

---

## Deployment

### Production

Deployed on **Vercel** (preferred for Next.js) or self-hosted with a Node.js server.

**Vercel setup:**
- Auto-deploys on push to `main`
- Preview deployments on PR branches
- Edge Middleware for auth + i18n
- Vercel Image Optimization pointed at Cloudflare R2
- Environment variables managed in Vercel Dashboard

**Self-hosted alternative:**
- `next build && next start` in Docker container
- Nginx reverse proxy with gzip + brotli
- Cloudflare proxied in front

### Cache Invalidation

On media data update (webhook from backend), the backend calls:
```
POST https://api.vercel.com/v1/integrations/deploy/...
// or
POST /api/revalidate?secret=...&slug=attack-on-titan&type=anime
```

Next.js `revalidatePath` and `revalidateTag` are used inside route handlers for on-demand ISR invalidation.
