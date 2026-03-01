# Keystatic Production Setup

A reference document for how Keystatic CMS is configured and used across this codebase.

---

## Table of Contents

1. [Dependencies](#dependencies)
2. [Core Configuration](#core-configuration)
3. [Collections](#collections)
4. [Admin Panel & API Routes](#admin-panel--api-routes)
5. [Authentication & Session Management](#authentication--session-management)
6. [Reading Content in Pages](#reading-content-in-pages)
7. [Rendering Document Fields](#rendering-document-fields)
8. [Next.js Configuration](#nextjs-configuration)
9. [Environment Variables](#environment-variables)
10. [Content Directory Structure](#content-directory-structure)
11. [TypeScript Types](#typescript-types)

---

## Dependencies

**File:** `package.json`

```json
"@keystatic/core": "^0.5.48",
"@keystatic/next": "^5.0.4"
```

- `@keystatic/core` — Core library for schema definitions, `createReader`, and `DocumentRenderer`
- `@keystatic/next` — Next.js integration for the admin UI and API route handler

---

## Core Configuration

**File:** `keystatic.config.ts`

```ts
import { config } from '@keystatic/core'

export default config({
  storage: {
    kind: 'github',
    repo: {
      owner: process.env.NEXT_PUBLIC_GITHUB_OWNER!,
      name: process.env.NEXT_PUBLIC_GITHUB_REPO!,
    },
  },
  ui: {
    brand: { name: 'Travel Blog CMS' },
  },
  collections: { posts, authors, expeditions, gallery },
})
```

**Storage backend:** GitHub. All content edits made via the admin panel are committed directly to the configured GitHub repository. The owner and repo are injected via `NEXT_PUBLIC_` environment variables so they are available on both server and client.

---

## Collections

All four collections use `format: { data: 'yaml' }` for metadata and store files under the `content/` directory.

### Posts (`posts`)

**Path:** `content/posts/*/`
**Slug field:** `title_en`

| Field | Type | Notes |
|---|---|---|
| `title_en` | slug | English title, used as the URL slug |
| `title_fr` | text | French title |
| `description_en` | multiline text | English excerpt |
| `description_fr` | multiline text | French excerpt |
| `content_en` | document | Rich text — formatting, dividers, links, images |
| `content_fr` | document | Rich text — formatting, dividers, links, images |
| `category` | select | `Destination`, `Culinary`, `Lifestyle`, `Tips & Hacks` |
| `image` | image | Featured image → `public/images/posts` |
| `author` | relationship | Links to `authors` collection |
| `date` | date | Publication date |
| `readTime` | integer | Estimated read time in minutes |
| `featured` | checkbox | Whether the post is featured |

**Image storage:**
```ts
image: fields.image({
  label: 'Featured Image',
  directory: 'public/images/posts',
  publicPath: '/images/posts/',
})
```

---

### Authors (`authors`)

**Path:** `content/authors/*`
**Slug field:** `name`

| Field | Type | Notes |
|---|---|---|
| `name` | slug | Author name, used as the collection key |
| `avatar` | image | Author photo → `public/images/authors` |
| `bio_en` | multiline text | English bio |
| `bio_fr` | multiline text | French bio |

---

### Expeditions (`expeditions`)

**Path:** `content/expeditions/*/`
**Slug field:** `title_en`

| Field | Type | Notes |
|---|---|---|
| `title_en` | slug | English title |
| `title_fr` | text | French title |
| `tagline_en` | text | Short English tagline |
| `tagline_fr` | text | Short French tagline |
| `heroImage` | image | Hero banner → `public/images/expeditions` |
| `description_en` | multiline text | English description |
| `description_fr` | multiline text | French description |
| `testimonials` | array | Array of `{ name, rating (1–5), text_en, text_fr }` |

> Additional fields (`duration`, `groupSize`, `difficulty`, `itinerary`, `pricing`) are defined but commented out in the config — ready to enable when needed.

---

### Gallery (`gallery`)

**Path:** `content/gallery/*/`
**Slug field:** `title_en`

| Field | Type | Notes |
|---|---|---|
| `title_en` | slug | English title |
| `title_fr` | text | French title |
| `src` | image | Gallery image → `public/images/gallery` |
| `category` | select | `Adventure`, `Nature`, `Culture`, `Urban` |
| `location_en` | text | English location label |
| `location_fr` | text | French location label |

---

## Admin Panel & API Routes

### Admin UI

**File:** `app/keystatic/[[...params]]/page.tsx`

```tsx
'use client'
import { makePage } from '@keystatic/next/ui/app'
import keystaticConfig from '../../../keystatic.config'

const KeystaticApp = makePage(keystaticConfig)

export default function KeystaticPage() {
  return (
    <KeystaticWrapper>
      <KeystaticApp />
    </KeystaticWrapper>
  )
}
```

The Keystatic admin UI is served at `/keystatic`. `makePage` from `@keystatic/next/ui/app` wires up the full CMS interface as a client component.

---

### API Route Handler

**File:** `app/api/keystatic/[[...params]]/route.ts`

```ts
import { makeRouteHandler } from '@keystatic/next/route-handler'
import keystaticConfig from '../../../../keystatic.config'

const { GET: _GET, POST } = makeRouteHandler({ config: keystaticConfig })

export async function GET(request: NextRequest) {
  const url = new URL(request.url)

  // Custom logout — clears GitHub OAuth tokens and iron-session cookie
  if (url.pathname === '/api/keystatic/github/logout') {
    // ... clears cookies and redirects to /admin/login
  }

  return _GET(request)
}

export { POST }
```

`makeRouteHandler` handles all Keystatic API operations (GitHub OAuth flow, content reads/writes). A custom `GET` intercept adds a logout handler that clears both Keystatic's GitHub OAuth cookies and the app's own `iron-session` cookie before redirecting to the login page.

---

## Authentication & Session Management

Keystatic's own GitHub OAuth is in use for CMS write access, but a separate custom auth layer guards the `/keystatic` route.

### Session Configuration

**File:** `lib/auth/session.ts`

```ts
import { getIronSession } from 'iron-session'

const sessionOptions = {
  cookieName: 'keystatic_auth_session',
  password: process.env.SESSION_SECRET!,
  cookieOptions: {
    secure: process.env.NODE_ENV === 'production',
    httpOnly: true,
    sameSite: 'lax' as const,
    maxAge: 60 * 60 * 24 * 7, // 7 days
  },
}

export function getSession(req, res) {
  return getIronSession(req, res, sessionOptions)
}
```

Uses `iron-session` for encrypted, server-side session cookies.

---

### Credential Validation

**File:** `lib/auth/credentials.ts`

Validates against `CMS_USERNAME` and `CMS_PASSWORD` environment variables. Simple string comparison — suitable for a single-admin MVP.

---

### Login Page & Actions

**Files:** `app/admin/login/page.tsx`, `app/admin/login/actions.ts`

- Login form at `/admin/login`
- Server action `login()` — validates credentials, creates session, redirects to `/keystatic`
- Server action `logout()` — destroys session
- Failed login redirects to `/admin/login?redirect=...` with an error message

---

## Reading Content in Pages

Content is read at build/request time using `createReader` from `@keystatic/core/reader`.

**Pattern used across all data-fetching pages:**

```ts
import { createReader } from '@keystatic/core/reader'
import keystaticConfig from '@/keystatic.config'

const reader = createReader(process.cwd(), keystaticConfig)

// Fetch all items in a collection
const posts = await reader.collections.posts.all()

// Fetch a single item by slug
const post = await reader.collections.posts.read(slug)

// Resolve a document field (returns an async iterator)
const content = await post.content_en()
```

**Pages that use `createReader`:**

| Page | Collections read |
|---|---|
| `app/page.tsx` | `expeditions`, `posts`, `authors` |
| `app/blog/page.tsx` | `posts` |
| `app/blog/[slug]/page.tsx` | `posts`, `authors` |
| `app/expedition/page.tsx` | `expeditions` |
| `app/gallery/page.tsx` | `gallery` |
| `app/api/search/expeditions/route.ts` | `expeditions` (language-aware search) |

---

## Rendering Document Fields

**File:** `components/blog/blog-post-client.tsx`

Keystatic `document` fields are rendered using `DocumentRenderer` from `@keystatic/core/renderer`:

```tsx
import { DocumentRenderer } from '@keystatic/core/renderer'

export function BlogPostClient({ post }) {
  return <DocumentRenderer document={post.content_en} />
}
```

`DocumentRenderer` handles all rich-text nodes (headings, bold, italic, links, images, dividers) that were configured in the `document` field schema.

---

## Next.js Configuration

**File:** `next.config.mjs`

```js
const nextConfig = {
  serverExternalPackages: ['@keystatic/core'],
  experimental: {
    outputFileTracingIncludes: {
      '/*': ['./content/**/*'],
    },
  },
}
```

- **`serverExternalPackages`** — Prevents `@keystatic/core` from being bundled by Webpack on the server, avoiding issues in serverless/edge environments.
- **`outputFileTracingIncludes`** — Ensures the `content/` directory (YAML and `.mdoc` files) is included in the Vercel deployment output. Without this, static content files would be missing at runtime.

---

## Environment Variables

| Variable | Required | Description |
|---|---|---|
| `NEXT_PUBLIC_GITHUB_OWNER` | Yes | GitHub username or org that owns the repo |
| `NEXT_PUBLIC_GITHUB_REPO` | Yes | GitHub repository name |
| `SESSION_SECRET` | Yes | Long random string for signing iron-session cookies |
| `CMS_USERNAME` | Yes | Admin username for the `/admin/login` page |
| `CMS_PASSWORD` | Yes | Admin password for the `/admin/login` page |
| `NODE_ENV` | Auto | Controls `secure` flag on session cookie (`true` in `production`) |

> `NEXT_PUBLIC_` variables are embedded into the client bundle at build time. Keep `SESSION_SECRET`, `CMS_USERNAME`, and `CMS_PASSWORD` server-side only (no `NEXT_PUBLIC_` prefix).

---

## Content Directory Structure

All Keystatic content lives under `content/` and is committed to the repository. The GitHub storage backend reads and writes files here directly.

```
content/
├── authors/
│   └── mayukh-chakraborty.yaml
├── posts/
│   ├── lille-the-city-of-olympic-dreams-and-sporting-legacy/
│   │   ├── index.yaml          # Metadata (title, date, category, etc.)
│   │   ├── content_en.mdoc     # English rich text
│   │   └── content_fr.mdoc     # French rich text
│   └── tesdt/
│       ├── index.yaml
│       ├── content_en.mdoc
│       └── content_fr.mdoc
├── expeditions/
│   ├── lille-i-love-you/
│   │   └── index.yaml
│   ├── vive-la-france/
│   ├── tasty-lille/
│   ├── walking-down-the-memory-lane-of-the-old-city/
│   └── spooky-lille/
└── gallery/
    ├── eura-creative/
    ├── interview-for-a-grant/
    ├── halloween-day-guests-2024/
    ├── festival-of-giants/
    ├── grand-place-at-night/
    ├── a-day-in-ypres/
    ├── hommage-to-fallen-soldiers/
    └── one-museum-in-paris/
```

**Post/expedition format:**
- `index.yaml` — All scalar fields (title, date, image path, etc.)
- `content_en.mdoc` / `content_fr.mdoc` — Document field content in Markdoc format

**Authors/gallery format:**
- Single `.yaml` file per item (no separate document files)

**Images** are stored in `public/images/<collection>/` and referenced by path in the YAML files.

---

## TypeScript Types

**File:** `lib/types.ts`

Manually maintained interfaces that mirror the Keystatic collection schemas. Used for type-safe data passing between server components (which call `createReader`) and client components.

| Type | Description |
|---|---|
| `Author` | Author profile with bilingual bio and avatar |
| `BlogPost` | Post metadata + bilingual content + category/date |
| `BlogPostWithAuthor` | `BlogPost` with the resolved `Author` object |
| `Expedition` | Expedition with hero image, descriptions, and testimonials array |
| `GalleryItem` | Gallery image with category and bilingual location |

> These types are separate from the Keystatic-inferred types. If you add or remove fields in `keystatic.config.ts`, update `lib/types.ts` accordingly.
