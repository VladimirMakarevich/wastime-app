# Web Site Architecture

The Wastime web site (*Wastime UI*) is an **Angular 21** application rendered with
**server-side rendering / static prerendering**. It serves the marketing landing page,
per-feature SEO landing pages, a markdown-driven blog, legal/support pages, and web access to
the product's account area.

For versions and package details, see [tech-stack.md](tech-stack.md#web-site).

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

```text
src/app/
  pages/            Route-level screens
    landing/            Main marketing landing
    feature-landings/   One SEO landing per product feature
    blog/               Blog index, post, tag/pagination views
    life-calendar/ · calendars/   Interactive web versions
    legal/              Privacy, Terms, Data Protection
    support/ · support-us/ · marketing/
    account/ · user/    Authenticated area
    error404/
  features/
    calendar-snake/     Interactive "calendar snake" mini-experience
  components/         Reusable UI (menu, footer, nav bars, breadcrumb,
                      share-buttons, download-app-modal, preloader, …)
  core/              Cross-cutting: services, guards, interceptors,
                     message-bus, models, pipes, helpers, constants, utils
  app.routes.ts      Route table (+ app.config.ts / app.config.server.ts)
```

The `core/` layer mirrors the app's conventions (a `message-bus`, `interceptors`, `guards`,
typed `services`) so patterns are consistent across the two Angular codebases.

---

## Routing & pages

Routing (`app.routes.ts`) is organized around SEO landing pages and content:

| Route group | Purpose |
| --- | --- |
| `/` , `/about` | Main landing (hero, philosophy, features, FAQ, CTA) |
| `/features/*` | Dedicated landing per feature: `life-calendar`, `time-analysis`, `life-expectancy`, `live-calculator`, `wasted-calculator`, `converter`, `questions` |
| `/blog/**` | Blog index, posts, tag and pagination routes |
| `/marketing`, `/support-us` | Campaign / support pages |
| `/privacy-policy`, `/terms-of-service`, `/data-protection` | Legal |
| `/contact`, `/support` | Support |
| `/account`, `/user/**` | Authenticated area |
| `/404` | Not found |

Two route guards shape access:

- **`stagingAccessGuard`** — gates the site on staging environments (behind a staging-access
  modal).
- **`redirectAuthenticatedGuard`** — routes signed-in users away from marketing entry points.

Legacy blog URLs are preserved via a `BLOG_REDIRECTS` table prepended to the routes, so old
links keep working (complemented by build-time redirect generation).

---

## Rendering strategy: SSR + prerendering

The site is built as a **prerendered (SSG) Angular app**:

- **`@angular/ssr` + `@angular/platform-server`** provide server rendering.
- `angular.json` configures a **server entry** (`src/main.server.ts`) and **prerender** using
  a routes file (`src/prerender-routes.txt`).
- That routes file is **generated** from static routes plus every blog route derived from the
  content manifest — so new posts automatically get prerendered pages.

```text
src/prerender-routes.txt   ← generated (static routes + blog routes from manifest)
        │
        ▼
ng build --configuration production   → prerenders each route to static HTML
        │
        ▼
dist/ (static HTML + hydrating Angular app)
```

The result: crawlers and first paint get real HTML; the Angular app then hydrates for
interactivity.

---

## Content & blog pipeline

The blog is **content-as-code**. Authors write Markdown with front-matter; a Node pipeline
compiles it into JSON assets and prerender routes.

```text
content/blog/posts/*.md   (Markdown + front-matter, + assets/)
        │
        │  validate-blog-content.mjs      (fail fast on bad content)
        ▼
   build-blog-content.mjs
     • gray-matter → parse front-matter
     • marked → Markdown to HTML
     • blog-linking.mjs → rewrite [[wiki-links]], normalize SEO hrefs
     • compute word count / reading time / pagination
        │
        ├──▶ src/assets/blog/manifest.json         (all published posts' metadata)
        └──▶ src/assets/blog/posts/<slug>.json      (full post + rendered HTML)
        │
        ├──▶ generate-og-images.mjs      → per-post Open Graph images (canvas)
        ├──▶ generate-redirects.mjs      → redirect rules
        └──▶ generate-prerender-routes.mjs → src/prerender-routes.txt
                                              (paginated blog + tag indexes)
```

At runtime the Angular blog components read the generated `manifest.json` and per-slug JSON —
there is **no CMS or database** in the request path, which keeps the site static and cheap to
host.

| Script | Responsibility |
| --- | --- |
| `validate-blog-content.mjs` | Validates front-matter/content before building |
| `build-blog-content.mjs` | Markdown → JSON (manifest + per-post) |
| `blog-linking.mjs` | Internal wiki-link rewriting and SEO href normalization |
| `generate-og-images.mjs` | Open Graph share images |
| `generate-redirects.mjs` | Redirect rules for moved/legacy URLs |
| `generate-prerender-routes.mjs` | Builds the prerender route list |
| `generate-seo-artifacts.mjs` | Sitemaps and SEO artifacts (host-targeted) |

---

## SEO artifacts

`generate-seo-artifacts.mjs` runs after the production build and is **target-aware**
(`--target=azure` or `--target=firebase`), emitting host-appropriate:

- **Sitemaps** covering all prerendered routes and blog posts.
- **Redirects** (from `generate-redirects.mjs`) wired into each host's config.
- **Open Graph images** and metadata for social sharing.
- Canonical URLs and per-page titles (route `data: { title }`).

---

## Authentication

The web site uses **Firebase Authentication** (`@firebase/auth`) — the *same* identity system
as the mobile app and backend, so a user's account is unified across surfaces. Sign-in powers
the `/account` and `/user` area, while `redirectAuthenticatedGuard` keeps authenticated users
out of marketing funnels. (Apple web sign-in is planned per the UI repo's auth docs.)

---

## Build & deployment

The site targets **two hosts** from one codebase:

```text
                 ┌───────────────────────────────┐
                 │  Content + Angular source     │
                 └───────────────┬───────────────┘
                                 │ validate → content:build →
                                 │ generate-prerender-routes → ng build (prod)
                                 │ → generate-seo-artifacts
                ┌────────────────┴─────────────────┐
                ▼                                   ▼
   build:prerender:firebase              build:prerender:azure
   (config: production,firebase)         (config: production)
                ▼                                   ▼
     Firebase Hosting                     Azure Static Web Apps
     (firebase.json, .firebaserc)         (staticwebapp.config.json)
```

| Target | Config | Command |
| --- | --- | --- |
| **Firebase Hosting** (primary) | `firebase.json`, `.firebaserc` | `npm run deploy` → `build:prerender:firebase` + `firebase deploy --only hosting` |
| **Azure Static Web Apps** | `staticwebapp.config.json` | `npm run build:prerender:azure` |

Both flows: validate content → build blog JSON → generate prerender routes → production
Angular build (prerender) → generate SEO artifacts for the chosen host.

---

## Shared with the rest of the system

- **API client** — like the mobile app, the site consumes the **generated Angular client**
  from the backend's `openapi/swagger.json` (`projects/waste-api-client`), so web ↔ API stays
  type-safe. See [backend architecture](architecture-backend.md#api-contract--client-generation).
- **Firebase identity** — one auth system across app, web, and API.
- **Conventions** — a shared `core/` shape (message-bus, guards, interceptors, services) keeps
  the two Angular codebases consistent.

---

## Quality & tooling

| Concern | Tooling |
| --- | --- |
| **Linting** | ESLint (`@angular-eslint`), Stylelint for SCSS |
| **Formatting** | Prettier |
| **Testing** | Karma + Jasmine (unit, with coverage), Angular e2e (with a CI configuration) |
| **Git hooks** | Husky (pre-commit lint/format) |
| **CI/CD** | GitHub Actions (build + deploy) |
