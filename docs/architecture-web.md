# Web Site Architecture

The Wastime web site is an **Angular 21** application rendered with **server-side rendering /
static prerendering**. It serves the marketing landing page, per-feature SEO landing pages, a
markdown-driven blog, legal/support pages, and web access to the product's account area.

> This document describes the site at the level of **roles, patterns, and pipeline stages**
> rather than concrete file, route, or script names. For versions and libraries, see
> [tech-stack.md](tech-stack.md#web-site).

---

## Role & goals

Unlike the offline-first app, the web site's job is **discovery and trust**: it must be fast,
crawlable, and shareable.

| Goal | Architectural consequence |
| --- | --- |
| **SEO / crawlability** | Pages are **prerendered to static HTML** (Angular SSR) with sitemaps and canonical URLs. |
| **Rich sharing** | **Open Graph images** are generated per page/post at build time. |
| **Content as code** | The blog is authored in **Markdown + front-matter**, validated and compiled in CI. |
| **Portability** | The same build deploys to **two hosts** (Firebase Hosting and Azure Static Web Apps). |
| **One identity, one contract** | Shares **Firebase auth** and the **generated API client** with the mobile app. |

---

## Application structure

The app is organized by role, mirroring the mobile app's conventions so patterns stay
consistent across the two Angular codebases:

| Area | Contents |
| --- | --- |
| **Route-level screens** | Marketing landing, per-feature SEO landings, blog views, interactive web tools, legal, support, the authenticated area, and a not-found page |
| **Interactive features** | Small self-contained interactive experiences |
| **Shared UI components** | Menu, footer, navigation bars, breadcrumb, share buttons, download-app prompt, preloader, and similar building blocks |
| **Cross-cutting core** | Services, route guards, HTTP interceptors, an internal event bus, models, pipes, helpers, and utilities |
| **App configuration** | The route table plus separate client and server application configs |

---

## Routing & pages

Routing is organized around SEO landing pages and content:

| Route group | Purpose |
| --- | --- |
| **Home / about** | Main landing (hero, philosophy, features, FAQ, CTA) |
| **Per-feature landings** | One dedicated SEO landing per product feature (life calendar, time analysis, life-expectancy, the calculators, converter, questions) |
| **Blog** | Index, posts, and tag / pagination routes |
| **Campaign & support** | Marketing and support pages |
| **Legal** | Privacy, Terms, Data Protection |
| **Authenticated area** | Account and profile |
| **Not found** | 404 |

Two route guards shape access:

- A **staging-access guard** gates the site on staging environments (behind a staging-access
  prompt).
- An **authenticated-redirect guard** routes signed-in users away from marketing entry points.

Legacy blog URLs are preserved via a **redirect table** prepended to the routes, complemented
by build-time redirect generation, so old links keep working.

---

## Rendering strategy: SSR + prerendering

The site is built as a **prerendered (SSG) Angular app**:

- **`@angular/ssr` + `@angular/platform-server`** provide server rendering.
- The build configures a **server entry** and prerenders the site using a **generated routes
  list**.
- That routes list is generated from static routes plus every blog route derived from the
  content manifest — so new posts automatically get prerendered pages.

```text
Generated route list   (static routes + blog routes from the content manifest)
        │
        ▼
Production build   → prerenders each route to static HTML
        │
        ▼
Static HTML + a hydrating Angular app
```

The result: crawlers and first paint get real HTML; the Angular app then hydrates for
interactivity.

---

## Content & blog pipeline

The blog is **content-as-code**. Authors write Markdown with front-matter; a Node pipeline
compiles it into JSON assets and prerender routes.

```text
Markdown posts   (front-matter + per-post assets)
        │
        │  validation      (fail fast on bad content)
        ▼
   Content build
     • gray-matter → parse front-matter
     • marked → Markdown to HTML
     • internal wiki-link rewriting + SEO href normalization
     • compute word count / reading time / pagination
        │
        ├──▶ posts manifest        (metadata for all published posts)
        └──▶ per-post JSON          (full post + rendered HTML)
        │
        ├──▶ per-post Open Graph images   (canvas)
        ├──▶ redirect rules
        └──▶ generated prerender route list   (paginated blog + tag indexes)
```

At runtime the Angular blog components read the generated manifest and per-post JSON — there is
**no CMS or database** in the request path, which keeps the site static and cheap to host.

| Pipeline stage | Responsibility |
| --- | --- |
| **Validation** | Validates front-matter / content before building |
| **Content build** | Markdown → JSON (manifest + per-post) |
| **Internal linking** | Wiki-link rewriting and SEO href normalization |
| **OG images** | Open Graph share images |
| **Redirects** | Redirect rules for moved / legacy URLs |
| **Prerender routes** | Builds the prerender route list |
| **SEO artifacts** | Sitemaps and SEO artifacts (host-targeted) |

---

## SEO artifacts

A build step runs after the production build and is **target-aware** (per host), emitting
host-appropriate:

- **Sitemaps** covering all prerendered routes and blog posts.
- **Redirects** wired into each host's configuration.
- **Open Graph images** and metadata for social sharing.
- Canonical URLs and per-page titles (carried as route metadata).

---

## Authentication

The web site uses **Firebase Authentication** (`@firebase/auth`) — the *same* identity system
as the mobile app and backend, so a user's account is unified across surfaces. Sign-in powers
the authenticated area, while the authenticated-redirect guard keeps signed-in users out of
marketing funnels. (Apple web sign-in is planned.)

---

## Build & deployment

The site targets **two hosts** from one codebase:

```text
                 ┌───────────────────────────────┐
                 │  Content + Angular source     │
                 └───────────────┬───────────────┘
                                 │ validate → build blog content →
                                 │ generate prerender routes → production build →
                                 │ generate SEO artifacts (per host)
                ┌────────────────┴─────────────────┐
                ▼                                   ▼
        Firebase Hosting                   Azure Static Web Apps
        (primary)                          (alternative target)
```

| Target | Notes |
| --- | --- |
| **Firebase Hosting** (primary) | Primary hosting + Firebase Auth |
| **Azure Static Web Apps** | Alternative deploy target |

Both flows share the same pipeline: validate content → build blog JSON → generate prerender
routes → production Angular build (prerender) → generate SEO artifacts for the chosen host.

---

## Shared with the rest of the system

- **API client** — like the mobile app, the site consumes the **generated Angular client**
  from the backend's OpenAPI specification, so web ↔ API stays type-safe. See
  [backend architecture](architecture-backend.md#api-contract--client-generation).
- **Firebase identity** — one auth system across app, web, and API.
- **Conventions** — a shared cross-cutting core (event bus, guards, interceptors, services)
  keeps the two Angular codebases consistent.

---

## Quality & tooling

| Concern | Tooling |
| --- | --- |
| **Linting** | ESLint (`@angular-eslint`), Stylelint for SCSS |
| **Formatting** | Prettier |
| **Testing** | Karma + Jasmine (unit, with coverage), Angular e2e (with a CI configuration) |
| **Git hooks** | Husky (pre-commit lint/format) |
| **CI/CD** | GitHub Actions (build + deploy) |
